---
layout: post
title: 翻译：Android MVC MVP MVVM比较
description: "Android MVC MVP MVVM比较"
modified: 2017-03-17
dev: [Android,translation]
type: dev
---

本文为翻译文章，这里是[原文链接](https://realm.io/news/eric-maxwell-mvc-mvp-and-mvvm-on-android/?utm_source=Android+Weekly&utm_campaign=4355a5dcbf-AndroidWeekly_242&utm_medium=email&utm_term=0_4eb677ad19-4355a5dcbf-337956589)

##### 背景

在过去几年中，将Android应用程序组织成逻辑组件的最佳实践方法已经发生了变化。 社区已经远离整体模型视图控制器（MVC）模式，倾向于更模块化，可测试的模式。

模型视图呈现器（MVP）和模型视图ViewModel（MVVM）是最广泛采用的两种替代方法，但开发人员通常纠结于哪一种更适合Android。 在过去一年里，有大量的博客文章强烈倡导这个反对那个，但通常这些变成对客观标准的争论。 本文力图客观地看待所有三种方法的价值和潜在问题，以便您可以为自己做出明智的决定而不是争论哪种方法更好。

为了帮助我们看清每个模式的行为，我们使用简单的Tic-Tac-Toe游戏（译注：井字棋游戏，在3x3的棋盘上，双方轮流落子，先将3枚棋子连成一线的一方获得胜利）。

demo源代码可以从[GitHub仓库](https://github.com/ericmaxwell2003/ticTacToe)获取，在代码中你可以check out当下阅读部分的分支（如git checkout mvc,git checkout mvp,git checkout mvvm)。

##### MVC

在宏观层面上将应用分层3组，模型（model），视图（View），控制器（Controller）各司其职。

###### Model

在我们的Tic-Tac-Toe应用中model层是指数据+状态+业务逻辑，它是我们应用的大脑，它不绑定到view或者controller，因为这个原因，它在许多上下文中是可重用的。

###### view

view是Model的表示，当用户与应用交互时，view负责呈现用户界面（UI）并与controller通信，在MVC结构中，Views通常是非常“无言的”，因为它们不知道底层模型，没有理解状态或当用户通过点击按钮，输入值等交户操作时做什么。这样的设计是基于它们知道的越少则模型的耦合越松散，因此它们改变会更灵活。

###### Controller

controller是把应用捆绑在一起的“胶”，它是应用程序中事件发生的主controller。当View告诉controller有用户点击按钮时，controller决定如何与model进行交互。基于model中的数据改变，controller可以决定适当的更新view状态，在此模式下的Android应用中，controller几乎总是相当于Activity或Fragment。



下图为Tic Tac Toe应用中对应的文件结构。

![android mvc]({{ site.url }}/images/android_post/android_mvc.png)

让我们更详细地检视controller。

```
public class TicTacToeActivity extends AppCompatActivity {

    private Board model;

    /* View Components referenced by the controller */
    private ViewGroup buttonGrid;
    private View winnerPlayerViewGroup;
    private TextView winnerPlayerLabel;

    /**
     * In onCreate of the Activity we lookup & retain references to view components
     * and instantiate the model.
     */
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.tictactoe);
        winnerPlayerLabel = (TextView) findViewById(R.id.winnerPlayerLabel);
        winnerPlayerViewGroup = findViewById(R.id.winnerPlayerViewGroup);
        buttonGrid = (ViewGroup) findViewById(R.id.buttonGrid);

        model = new Board();
    }

    /**
     * Here we inflate and attach our reset button in the menu.
     */
    @Override
    public boolean onCreateOptionsMenu(Menu menu) {
        MenuInflater inflater = getMenuInflater();
        inflater.inflate(R.menu.menu_tictactoe, menu);
        return true;
    }
    /**
     *  We tie the reset() action to the reset tap event.
     */
    @Override
    public boolean onOptionsItemSelected(MenuItem item) {
        switch (item.getItemId()) {
            case R.id.action_reset:
                reset();
                return true;
            default:
                return super.onOptionsItemSelected(item);
        }
    }

    /**
     *  When the view tells us a cell is clicked in the tic tac toe board,
     *  this method will fire. We update the model and then interrogate it's state
     *  to decide how to proceed.  If X or O won with this move, update the view
     *  to display this and otherwise mark the cell that was clicked.
     */
    public void onCellClicked(View v) {

        Button button = (Button) v;

        int row = Integer.valueOf(tag.substring(0,1));
        int col = Integer.valueOf(tag.substring(1,2));

        Player playerThatMoved = model.mark(row, col);

        if(playerThatMoved != null) {
            button.setText(playerThatMoved.toString());
            if (model.getWinner() != null) {
                winnerPlayerLabel.setText(playerThatMoved.toString());
                winnerPlayerViewGroup.setVisibility(View.VISIBLE);
            }
        }

    }

    /**
     * On reset, we clear the winner label and hide it, then clear out each button.
     * We also tell the model to reset (restart) it's state.
     */
    private void reset() {
        winnerPlayerViewGroup.setVisibility(View.GONE);
        winnerPlayerLabel.setText("");

        model.restart();

        for( int i = 0; i < buttonGrid.getChildCount(); i++ ) {
            ((Button) buttonGrid.getChildAt(i)).setText("");
        }
    }
}
```

###### 评价

MVC在分离model和view方面做得不够好，当然该模型可以轻松地测试，因为它不依赖于任何东西并且view在单元测试级别没有更多的测试。 然而Controller有一些问题。

###### Controller的问题

* 可测试性 - Controller与Android APIs紧密绑定因此很难进行单元测试。
* 模块化&灵活性 - Controller与views紧耦合，它也可能是view的扩展，如果改变view，则不得不改变controller。
* 维护 - 随着时间的推移，特别是在具有[anemic models](https://martinfowler.com/bliki/AnemicDomainModel.html)的应用中，越来越多的代码开始向controllers转移，使其膨胀和脆弱。

我们如何解决这些问题？MVP来了！



#### MVP

MVP打破了controller，使得正常下view／activity耦合能够发生而不会影响其余的“controller”的职责，更多请见下文，但让我们从它和MVC的职责的常见定义对比开始。

###### Model

和MVC一样，没有任何不同。

###### View

唯一的变化是Activity／Fragment被认为是视图的一部分，一个好的实践方式是Activity实现一个view接口以便presenter有一个接口去完成，这样做消除了它耦合到任何特定的view，并允许对view的模拟实现进行简单的单元测试。

###### Presenter

除了它只是一个接口而不绑定到view外，它本质上是来自于MVC的controller。它解决了我们MVC模式的可测试性以及模块化／灵活性问题。事实上，MVP理想主义者会认为presenter永远不应该有任何Android APIs或代码的引用。

让我们再一次检查该模式在我们应用中的实现图。

![android mvp]({{ site.url }}/images/android_post/android_mvp.png)

下文展示更详细的Presenter，你将会注意到的第一件事是每一个action的意图都更简单和清楚，它只是告诉view要显示什么而非如何显示。

```
public class TicTacToePresenter implements Presenter {

    private TicTacToeView view;
    private Board model;

    public TicTacToePresenter(TicTacToeView view) {
        this.view = view;
        this.model = new Board();
    }

    // Here we implement delegate methods for the standard Android Activity Lifecycle.
    // These methods are defined in the Presenter interface that we are implementing.
    public void onCreate() { model = new Board(); }
    public void onPause() { }
    public void onResume() { }
    public void onDestroy() { }

    /** 
     * When the user selects a cell, our presenter only hears about
     * what was (row, col) pressed, it's up to the view now to determine that from
     * the Button that was pressed.
     */
    public void onButtonSelected(int row, int col) {
        Player playerThatMoved = model.mark(row, col);

        if(playerThatMoved != null) {
            view.setButtonText(row, col, playerThatMoved.toString());

            if (model.getWinner() != null) {
                view.showWinner(playerThatMoved.toString());
            }
        }
    }

    /**
     *  When we need to reset, we just dictate what to do.
     */
    public void onResetSelected() {
        view.clearWinnerDisplay();
        view.clearButtons();
        model.restart();
    }
}
```

我们创建一个Activity实现的接口去让它在不绑定activity到presenter的情况下工作。在这个测试中，我们激昂创建一个基于这个接口的模型去测试view和presenter的交互。

```
public interface TicTacToeView {
    void showWinner(String winningPlayerDisplayLabel);
    void clearWinnerDisplay();
    void clearButtons();
    void setButtonText(int row, int col, String text);
}
```

##### 评价

这是一个更干净的模式，我们可以轻松地对presenter逻辑进行单元测试，因为它依赖于任何Android特定的views和APIs，并且还允许我们使用其它view，只要该view实现了TicTacToeView接口。

###### Presenter的问题

* 可维护性 - Presenters，就像Controllers一样，随着时间的推移，容易聚集越来越多的业务逻辑，在某个时间段，开发人员经常发现自己代码中拥有难以分解的大而笨重的presenters。

当然，细心的开发者能预防这种情况的出现，随着应用的变化随时提高警惕努力防止presenter的过重， 但是，MVVM可以让事情变得更简单。

##### MVVM

Android中使用[数据绑定](https://developer.android.com/topic/libraries/data-binding/index.html)的MVVM具有更易于测试和模块化的优点，同时还减少了我们为连接view、model而编写的代码的数量。

让我们来看看MVVM部分。

###### Model

和MVC一样，没有任何改变。

###### View

view通过viewModel以灵活的方式绑定到可见的变量和暴露出来的actions。 下文将揭示更多。

###### ViewModel

ViewModel负责包装model和准备view需要的可观察数据，同时它还提供给view传递事件到model的钩子，然而ViewModel并不会和view捆绑。

下图为Tic Tac Toe关系图。

![android mvvm]({{ site.url }}/images/android_post/android_mvvm.png)

让我们从ViewModel开始仔细看看一些变动的部分。

```
public class TicTacToeViewModel implements ViewModel {

    private Board model;

    /* 
     * These are observable variables that the viewModel will update as appropriate
     * The view components are bound directly to these objects and react to changes
     * immediately, without the ViewModel needing to tell it to do so. They don't
     * have to be public, they could be private with a public getter method too.
     */
    public final ObservableArrayMap<String, String> cells = new ObservableArrayMap<>();
    public final ObservableField<String> winner = new ObservableField<>();

    public TicTacToeViewModel() {
        model = new Board();
    }

    // As with presenter, we implement standard lifecycle methods from the view
    // in case we need to do anything with our model during those events.
    public void onCreate() { }
    public void onPause() { }
    public void onResume() { }
    public void onDestroy() { }

    /**
     * An Action, callable by the view.  This action will pass a message to the model
     * for the cell clicked and then update the observable fields with the current
     * model state.
     */
    public void onClickedCellAt(int row, int col) {
        Player playerThatMoved = model.mark(row, col);
        cells.put("" + row + col, playerThatMoved == null ? 
                                                     null : playerThatMoved.toString());
        winner.set(model.getWinner() == null ? null : model.getWinner().toString());
    }

    /**
     * An Action, callable by the view.  This action will pass a message to the model
     * to restart and then clear the observable data in this ViewModel.
     */
    public void onResetSelected() {
        model.restart();
        winner.set(null);
        cells.clear();
    }

}
```

下面这个view的片段展示变量和actions是如何被绑定的。

```
<!-- 
    With Data Binding, the root element is <layout>.  It contains 2 things.
    1. <data> - We define variables to which we wish to use in our binding expressions and 
                import any other classes we may need for reference, like android.view.View.
    2. <root layout> - This is the visual root layout of our view.  This is the root xml tag in the MVC and MVP view examples.
-->
<layout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    xmlns:app="http://schemas.android.com/apk/res-auto">

    <!-- We will reference the TicTacToeViewModel by the name viewModel as we have defined it here. -->
    <data>
        <import type="android.view.View" />
        <variable name="viewModel" type="com.acme.tictactoe.viewmodel.TicTacToeViewModel" />
    </data>
    <LinearLayout...>
        <GridLayout...>
            <!-- onClick of any cell in the board, the button clicked will invoke the onClickedCellAt method with its row,col -->
            <!-- The display value comes from the ObservableArrayMap defined in the ViewModel  -->
            <Button
                style="@style/tictactoebutton"
                android:onClick="@{() -> viewModel.onClickedCellAt(0,0)}"
                android:text='@{viewModel.cells["00"]}' />
            ...
            <Button
                style="@style/tictactoebutton"
                android:onClick="@{() -> viewModel.onClickedCellAt(2,2)}"
                android:text='@{viewModel.cells["22"]}' />
        </GridLayout>

        <!-- The visibility of the winner view group is based on whether or not the winner value is null.
             Caution should be used not to add presentation logic into the view.  However, for this case
             it makes sense to just set visibility accordingly.  It would be odd for the view to render
             this section if the value for winner were empty.  -->
        <LinearLayout...
            android:visibility="@{viewModel.winner != null ? View.VISIBLE : View.GONE}"
            tools:visibility="visible">

            <!-- The value of the winner label is bound to the viewModel.winner and reacts if that value changes -->
            <TextView
                ...
                android:text="@{viewModel.winner}"
                tools:text="X" />
            ...
        </LinearLayout>
    </LinearLayout>
</layout>
```

*提示：在上文的例子中大量使用tools属性，用来进行显示的值和可见性设置。 如果你不设置这些，在设计时可能很难看到事实效果。*

*关于Android中数据绑定的附注， 这只是数据绑定技术的冰山一角。我强烈建议你查看[Android Data Binding](https://developer.android.com/topic/libraries/data-binding/index.html)的文档，以便学习更多这一强大的工具，本页底部还有一个链接，指向Google Android Architecture Blueprints项目页面，以便你了解更多的MVVM和数据绑定的例子*

###### 评价

单元测试现在更容易了，因为你对view没有依赖性，测试时，你只需要验证在model更改时可观察的变量是否有对应的变化，没有必要模拟view测试。

###### MVVM的问题

* 可维护性 - 由于views可以绑定到变量和表达式，无关的表示逻辑可能随着时间的推移使我们XML代码数量增加，要避免这种情况，请始终直接从ViewModel获取值，而不是尝试在views绑定表达式中计算或导出它们。 这样方式，计算可以被适当地单元测试。

##### 结论

MVP和MVVM在应用模块化，单一化组件方面比MVC做得更好，但是它们也让你的应用更复杂，对于只有一两个页面的简单应用来说，MVC可能更合适，具有数据绑定的MVVM是有吸引力的，因为它遵循更具反应性的编程模型并产生更少的代码。

那么哪一种模式更适合你？如果你在MVP和MVVM之间选择，很多决定来自个人偏好，但看到它们各自的形式将帮助你了解的它们的优势并作出权衡。

如果你有兴趣看更多的MVP和MVVM在实践中的例子，我建议您查看[Google Architecture Blueprints](https://github.com/googlesamples/android-architecture)项目。 还有许多博客文章也都深入探讨正确的MVP实现方式。





















































































