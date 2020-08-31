### Drag And Drop

1. 什么是 Press-Drag-Release Gesture(姿势, 手势) ?  
2. **是用户按下鼠标按钮，拖动鼠标与按下的动作按钮，然后松开按钮。手势可以在场景或节点上启动**。几个节点和场景可参与单一的按-拖-释放手势。该手势能够生成不同类型的事件并将这些事件交付给不同的节点。所生成的事件和节点的类型事件取决于手势的目的。一个节点可以拖动不同的目的:
   1. 您可能希望通过拖动节点的边界来改变节点的形状，或者通过拖动节点到新的位置来移动节点。在这种情况下，手势只涉及到一个节点:启动手势的节点。
   2. 您可能希望拖动一个节点并将其拖放到另一个节点上，以某种方式连接它们，例如，用流程图中的符号连接两个节点。在本例中，拖动手势涉及多个节点。将源节点拖放到目标节点时，将发生一个操作。
   3. 您可以拖动一个节点并将其拖放到另一个节点上，以便将数据从源节点传输到目标节点。在本例中，拖动手势涉及多个节点。在删除源节点时发生数据传输。
3. JavaFX支持三种类型的拖动手势:
   1. 一个简单的按-拖-放手势
   2. 一个完整的按-拖-释放手势
   3. 拖放手势
4. 本章将主要关注第三种手势:拖放手势。要全面了解拖放手势，必须了解前两种手势。我将通过每种类型的一个简单示例简要讨论前两种类型的手势。

##### A Simple Press-Drag-Release Gesture

1. 简单的按-拖-放手势是默认的拖动手势. 当拖动手势只涉及到一个节点(即启动该手势的节点)时使用。在拖动手势期间，所有的MouseDragEvent类型(输入鼠标拖拽、拖动鼠标拖拽、退出鼠标拖拽、释放鼠标拖拽)都只传递给手势源节点。在本例中，当按下鼠标按钮时，将选择最上面的节点，并将所有后续鼠标事件交付给该节点，直到释放鼠标按钮为止。当将鼠标拖动到另一个节点时，启动手势的节点仍然在光标下，因此，在鼠标按钮被释放之前，其他节点不会接收事件。

2. 清单26-1中的程序演示了简单的按-拖-释放手势。它增加了两个textfield到一个场景:一个称为源节点，另一个称为目标节点。将事件处理程序添加到两个节点。目标节点添加MouseDragEvent处理程序来检测其上的任何鼠标拖动事件。运行程序，按下源节点上的鼠标按钮，将其拖到目标节点上，最后释放鼠标按钮。下面的输出显示源节点接收所有鼠标拖动事件。目标节点不接收任何鼠标拖动事件。这是一个简单的按下-拖放-释放手势的情况，其中初始化拖放手势的节点接收所有鼠标拖动事件。

3. ```java
   listing 26-1
   
   public class SimplePressDragRelease extends Application
   {
       TextField sourceFld = new TextField("Source Node");
       TextField targetFld = new TextField("Target node");
   
       public static void main(String[] args)
       {
           Application.launch(args);
       }
   
       @Override
       public void start(Stage stage)
       {
           // Build the UI
           GridPane root = getUI();
   
           // Add event handlers
           this.addEventHanders();
   
           Scene scene = new Scene(root);
           stage.setScene(scene);
           stage.setTitle("A simple press-drag-release gesture");
           stage.show();
       }
   
       private GridPane getUI()
       {
           GridPane pane = new GridPane();
           pane.setHgap(5);
           pane.setVgap(20);
           pane.addRow(0, new Label("Source Node:"), sourceFld);
           pane.addRow(1, new Label("Target Node:"), targetFld);
           return pane;
       }
   
       private void addEventHanders()
       {
           // Add mouse event handlers for the source
           sourceFld.setOnMousePressed(e -> print("Source: pressed"));
           sourceFld.setOnMouseDragged(e -> print("Source: dragged"));
           sourceFld.setOnDragDetected(e -> print("Source: dragged detected"));
           sourceFld.setOnMouseReleased(e -> print("Source: released"));
   
           // Add mouse event handlers for the target
           targetFld.setOnMouseDragEntered(e -> print("Target: drag entered"));
           targetFld.setOnMouseDragOver(e -> print("Target: drag over"));
           targetFld.setOnMouseDragReleased(e -> print("Target: drag released"));
           targetFld.setOnMouseDragExited(e -> print("Target: drag exited"));
       }
   
       private void print(String msg)
       {
           System.out.println(msg);
       }
   }
   ```

注意，drag-detected 事件是在拖动鼠标之后生成的。MouseEvent对象有一个dragDetect标志，可以在mouse-pressed 和mouse-dragged 事件中设置。如果将其设置为true，则生成的后续事件是drag-detected 事件。默认情况下是在mouse-dragged 事件之后生成它。如果希望在mouse-pressed 事件(而不是鼠标拖动事件)之后生成它，则需要修改事件处理程序:

```java
sourceFld.setOnMousePressed(e -> {
    print("Source: Mouse pressed");
    // Generate drag detect event after the current mouse pressed event
    e.setDragDetect(true);
});
sourceFld.setOnMouseDragged(e -> {
    print("Source: Mouse dragged");
    // Suppress the drag detected default event generation after mouse dragged
    e.setDragDetect(false);
});
```



##### A Full Press-Drag-Release Gesture  

当拖动手势的源节点接收到drag-detected 的事件时，您可以通过调用源节点上的startFullDrag()方法来启动完整的press-dragrelease手势。startFullDrag()方法存在于节点和场景类中，允许您为一个节点和场景启动一个完整的press-drag-release手势。在此讨论中，我将只使用术语节点。

Tip:  **startFullDrag()方法只能从拖放检测到的事件处理程序中调用。从任何其他位置调用这个方法抛出IllegalStateException**

您需要再进行一次设置，才能看到完整的“press-drag-release”手势的实际效果。当发生拖动时，拖动手势的源节点仍将接收所有mouse-drag 事件，因为它位于光标下。您需要将手势源的mouseTransparent属性设置为false，这样它下面的节点就会被选中，mouse-drag  事件就会被传递到那个节点。在mouse-pressed  事件中将此属性设置为true，在mouse-released  事件中将其设置为false

清单26-2中的程序演示了一个完整的按-拖-释放手势。该程序与清单26-1中显示的程序类似，除了以下情况:

1. 在源节点mouse-pressed的事件处理程序中，源节点的mouseTransparent属性被设置为false。在mouse-released 事件处理程序中，它被设为true。

2. 在drag-detected  事件处理程序中，在源节点上调用startFullDrag()方法。

运行程序，按下源节点上的鼠标按钮，将其拖到目标节点上，最后释放鼠标按钮。下面的输出显示，当鼠标在其边界内拖动时，目标节点接收mouse-drag   事件。这是完全按-拖-释放手势的情况，发生鼠标拖动的节点接收mouse-drag   事件

```java
public class FullPressDragRelease extends Application
{
    TextField sourceFld = new TextField("Source Node");
    TextField targetFld = new TextField("Target node");

    public static void main(String[] args)
    {
        Application.launch(args);
    }

    @Override
    public void start(Stage stage)
    {
        // Build the UI
        GridPane root = getUI();
        // Add event handlers
        this.addEventHanders();
        Scene scene = new Scene(root);
        stage.setScene(scene);
        stage.setTitle("A full press-drag-release gesture");
        stage.show();
    }

    private GridPane getUI()
    {
        GridPane pane = new GridPane();
        pane.setHgap(5);
        pane.setVgap(20);
        pane.addRow(0, new Label("Source Node:"), sourceFld);
        pane.addRow(1, new Label("Target Node:"), targetFld);
        return pane;
    }

    private void addEventHanders()
    {
        // Add mouse event handlers for the source
        sourceFld.setOnMousePressed(e ->
        {
            // Make sure the node is not picked
            sourceFld.setMouseTransparent(true);
            print("Source: Mouse pressed");
        });
        sourceFld.setOnMouseDragged(e -> print("Source: Mouse dragged"));
        sourceFld.setOnDragDetected(e ->
        {
            // Start a full press-drag-release gesture
            sourceFld.startFullDrag();
            print("Source: Mouse dragged detected");
        });
        sourceFld.setOnMouseReleased(e ->
        {
            // Make sure the node is picked
            sourceFld.setMouseTransparent(false);
            print("Source: Mouse released");
        });
        // Add mouse event handlers for the target
        targetFld.setOnMouseDragEntered(e -> print("Target: drag entered"));
        targetFld.setOnMouseDragOver(e -> print("Target: drag over"));
        targetFld.setOnMouseDragReleased(e -> print("Target: drag released"));
        targetFld.setOnMouseDragExited(e -> print("Target: drag exited"));
    }

    private void print(String msg)
    {
        System.out.println(msg);
    }
}
```



##### A Drag-and-Drop Gesture  

第三种类型的拖放手势称为Drag-and-Drop手势，它是一种用户操作，将鼠标移动与按下的鼠标按钮结合在一起。它用于将数据从手势源传输到手势目标。Drag-and-Drop手势允许从:

1.  One node to another node   一个节点到另外一个节点
2.  A node to a scene   一个节点到场景
3.  One scene to another scene   一个场景到另外一个场景
4.  A scene to a node   一个场景到节点

源和目标可以在同一个Java或JavaFX应用程序中，也可以在两个不同的Java或JavaFX应用程序中。一个JavaFX应用程序和一个本地应用程序也可能参与到手势，例如: 

1. 您可以拖动文本从Microsoft Word应用程序到JavaFX应用程序填充一个文本区域，反之亦然。
2. 您可以拖动图像文件从Windows资源管理器，并将其拖放到ImageView JavaFX应用程序。ImageView可以显示图像。
3. 您可以拖动文本文件从Windows资源管理器，并将其拖放到TextArea JavaFX应用程序。文本区域将读取文件并显示其内容。

**执行拖放手势需要几个步骤:**

	1. 在节点上按下鼠标按钮。
 	2. 按下按钮拖动鼠标。3
 	3. 节点接收到拖放检测到的事件。
 	4. 通过调用startDragAndDrop()在节点上启动一个拖放手势方法，使节点a成为手势源。来自源节点的数据被放置在一个dragboard  中。
 	5. 一旦系统切换到拖放手势，它就会停止传递MousEvents并开始交付DragEvents。
 	6. 手势源被拖到潜在的手势目标上。潜在的手势目标检查是否接受放置在dragboard  中的数据。如果它接受数据，它可能成为实际的手势目标。节点指示是否在其DragEvent处理程序之一中接受数据。
 	7. 用户释放手势目标上按下的按钮，并发送拖放事件。
 	8. 手势目标使用来自dragboard  的数据。
 	9. 一个拖放事件被发送到手势源，表示拖放手势已经完成。

