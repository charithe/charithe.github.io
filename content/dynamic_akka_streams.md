Title: Dynamic Akka Streams Using Stage Actors
Date: 2016-05-15 17:20
Category: Programming
Tags: Akka, Actors, Scala

Most stream processing frameworks require the implementer to define the processing logic using some sort of a DSL -- which then gets compiled and materialised into the framework's primitives at run time. The user has very little control over the pipeline once it starts executing. Any changes to the processing stages requires a re-compilation and a re-deployment. Getting around this limitation is not impossible but requires inelegant and costly operations such as polling an external service or a data store for updates.

[Akka Streams](http://doc.akka.io/docs/akka/2.4.4/scala/stream/#streams-scala) are materialised and run on top of actors but the DSL (understandably) completely hides this fact from the implementers. Letting users directly interact with actors could lead to inconsistent behaviour and even destabilise the entire pipeline. However,  in the recent releases of Akka streams, users are now able to access a so-called stage actor from within their custom graph stages. Stage actors are not fully-fledged actors and only support a subset of operations on them . This allows the framework to deal with the issues of maintaining consistency and thread-safety of the internal state during interactions with the stage actor.

To illustrate, I built a very rudimentary keyword-based tagging flow that can receive updates to keywords and tags while the system is running. `TagStage` is a custom graph stage that registers its' stage actor with an external coordinator actor on startup. The coordinator is then able to send messages to the stage actor -- which updates the internal map containing keywords and tags based on the received messages. When the internal state change is applied, the next element passed into the stage will be processed using the newly updated map -- thus changing the output of the system dynamically.

The `TagStage` implementation is as follows:


```scala
import akka.actor.ActorRef
import akka.stream.{Attributes, FlowShape, Inlet, Outlet}

import scala.collection.mutable
import akka.stream.stage.{GraphStage, GraphStageLogic, InHandler, OutHandler}

class TagStage(stageCoordinator: ActorRef, initialKeywordsMap: Map[String, Set[String]]) extends GraphStage[FlowShape[String, Set[String]]] {
  val in: Inlet[String] = Inlet("tag-stage-in")
  val out: Outlet[Set[String]] = Outlet("tag-stage-out")

  override def shape: FlowShape[String, Set[String]] = FlowShape(in, out)

  override def createLogic(inheritedAttributes: Attributes): GraphStageLogic =  new GraphStageLogic(shape) {
    implicit def self = stageActor.ref

    var keywordsMap = mutable.HashMap(initialKeywordsMap.toSeq:_*)

    setHandler(in, new InHandler {
      override def onPush(): Unit = {
        // Grab the next message from the inlet
        val msg = grab(in)
        // Tokenize the message and find all applicable tags
        val tags = msg.split("\\W+").collect {
          case flaggedWord if keywordsMap.contains(flaggedWord) => keywordsMap(flaggedWord)
        }.flatten

        if(tags.isEmpty){
          // if there are no tags to output, request next item from upstream
          tryPull(in)
        } else {
          // push the tags downstream
          push(out, Set(tags:_*))
        }
      }
    })

    setHandler(out, new OutHandler {
      // Request next item from upstream
      override def onPull(): Unit = tryPull(in)
    })

    override def preStart(): Unit = {
      // Register the stage actor with the coordinator actor
      val thisStageActor = getStageActor(messageHandler).ref
      stageCoordinator ! StageCoordinator.Register(thisStageActor)
    }

    private def messageHandler(receive: (ActorRef, Any)): Unit = {
      receive match {
        case (_, StageCoordinator.AddKeywordTags(keyword, tags)) => {
          // Add a new set of tags for the keyword
          val newTags = keywordsMap.get(keyword).map(_ ++ tags).getOrElse(tags)
          keywordsMap.update(keyword, newTags)
        }

        case (_, StageCoordinator.RemoveKeyword(keyword)) => {
          // Remove the keyword
          keywordsMap.remove(keyword)
        }
      }
    }
  }
}
```


The coordinator actor just needs to store the references to the registered stage actors and forward any received messages to them.


```
import akka.actor.{Actor, ActorLogging, ActorRef, Props, Terminated}
import com.github.charithe.akka.streams.StageCoordinator.{Register, TagCommand}

import scala.collection.mutable

class StageCoordinator extends Actor with ActorLogging {
  val stageActors: mutable.Set[ActorRef] = mutable.HashSet.empty

  override def receive = {
    case Register(actorRef) => {
      log.info("Registering stage actor [{}]", actorRef)
      stageActors.add(actorRef)
      context.watch(actorRef)
    }

    case Terminated(actorRef) => {
      log.info("Stage actor terminated [{}]", actorRef)
      stageActors.remove(actorRef)
      context.unwatch(actorRef)
    }

    case cmd: TagCommand => {
      log.info("Received command: [{}]", cmd)
      stageActors.foreach(_ ! cmd)
    }
  }
}

object StageCoordinator {

  final case class Register(actorRef: ActorRef)

  sealed trait TagCommand extends Product with Serializable

  final case class AddKeywordTags(keyword: String, tags: Set[String]) extends TagCommand

  final case class RemoveKeyword(keyword: String) extends TagCommand

  def props = Props[StageCoordinator]

}
```


Finally, the entire pipeline can be tested using the streams testkit.


```
import akka.actor.ActorSystem
import akka.stream.ActorMaterializer
import akka.stream.scaladsl.{Flow, Keep}
import akka.stream.testkit.scaladsl.{TestSink, TestSource}
import akka.testkit.{ImplicitSender, TestKit}
import org.scalatest.{BeforeAndAfterAll, FunSuiteLike, Matchers}

class TagStageTest(_system: ActorSystem) extends TestKit(_system) with ImplicitSender with FunSuiteLike with Matchers with BeforeAndAfterAll  {
  val TestMessage = "akka is a toolkit and runtime for building highly concurrent, distributed, and resilient message-driven applications on the JVM"
  implicit val materializer = ActorMaterializer()

  def this() = this(ActorSystem("tag-stage-test"))

  override def afterAll(): Unit = {
    TestKit.shutdownActorSystem(system)
  }

  test("Dynamic stage") {
    // Create the tag stage with testActor as the coordinator actor
    val tagStage = Flow.fromGraph(new TagStage(testActor, Map("akka" -> Set("scala", "actors"))))

    // Create the flow
    val (src, snk) = TestSource.probe[String]
      .via(tagStage)
      .toMat(TestSink.probe)(Keep.both)
      .run()

    // expect registration and subscription start messages
    val stageActorRegistration = expectMsgType[StageCoordinator.Register]
    val subscription = snk.expectSubscription()

    // Send a message through the pipeline
    subscription.request(1)
    src.sendNext(TestMessage)

    val tagResult1 = snk.expectNext()
    tagResult1 should have size 2
    tagResult1 should contain ("scala")
    tagResult1 should contain ("actors")

    // Update the keyword tag set through the stage actor
    stageActorRegistration.actorRef ! StageCoordinator.AddKeywordTags("akka", Set("jvm", "message-driven"))

    // Send the message through the pipeline again
    subscription.request(1)
    src.sendNext(TestMessage)

    val tagResult2 = snk.expectNext()
    tagResult2 should have size 4
    tagResult2 should contain ("scala")
    tagResult2 should contain ("actors")
    tagResult2 should contain ("jvm")
    tagResult2 should contain ("message-driven")

    // Remove the keyword through the stage actor
    stageActorRegistration.actorRef ! StageCoordinator.RemoveKeyword("akka")

    // Send the message through the pipeline again
    subscription.request(1)
    src.sendNext(TestMessage)

    // Since there are no keywords to match, nothing should come out
    snk.expectNoMsg()

    // Shutdown the flow
    src.sendComplete()
    snk.expectComplete()
  }
}
```


Source code for the project can be found [here](https://github.com/charithe/akka-stage-actor)
