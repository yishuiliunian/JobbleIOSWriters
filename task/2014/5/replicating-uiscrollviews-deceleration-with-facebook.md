Replicating UIScrollView’s deceleration with Facebook Pop
使用FaceceBook 的Pop框架替换UIScrollView的减速动画
====

Earlier this week Ole Begemann wrote a [really great tutorial on how UIScrollViews work](http://oleb.net/blog/2014/04/understanding-uiscrollview/), and to explain it effectively, he even created [a really simple scroll view, written from scratch.](https://github.com/ole/CustomScrollView)
这个星期 Ole Begemann 写了一篇[很棒的关于UIScrollView 是如何工作的教程](http://oleb.net/blog/2014/04/understanding-uiscrollview/)，为了更有效地描述这个UIScrollView的工作机制，他甚至还[自己从头开始写了一个简单的ScrollView。](https://github.com/ole/CustomScrollView)

The setup was quite simple: Use a UIPanGestureRecognizer, and change the bounds’ origin in response to translation of the pan gesture.
这个过程其实非常简单， 用一个`UIPanGestureRecognizer`， 然后通过修改界面的bounds原点 来对滑动手势的平移做出响应。

Extending Ole’s custom scroll view to incorporate UIScrollView’s inertial scrolling seemed like a natural extension, and with Facebook’s recent release of Pop, I thought it writing a decelerating custom scroll view would be an interesting weekend project.
我们很容易联想到可以在Ole的自定义ScrollView的基础上添加`UIScrollview`的惯性滑动行为，并且随着最近Facebook发布了POP动画引擎，让我觉得写出一个自然减速的自定义ScrollView会是一个很有趣的周末项目。

In case you’re unaware, [Pop is Facebook’s recently open-sourced animation framework](http://iosdevtips.co/post/84160910513/exploring-facebook-pop) that powers the animations in Paper for iOS. The Pop API looks mostly like CoreAnimation, so it isn’t that difficult to learn.
你可能不是很清楚，[POP 是Facebook最近开源的动画引擎框架](http://iosdevtips.co/post/84160910513/exploring-facebook-pop)，它是Paper for iOS里所有动画背后的动画引擎。Pop的API看起来很像`CoreAnimation`，所以学起来并不难。

Two of the best things about Pop:
关于为什么使用Pop的两点最好的说明：

1. It gives you spring and decay animations out of the box
1. 它可以很容易地实现弹力和减速动画

2. You can animate any property on any NSObject using Pop, not just the ones that are declared animatable in UIKit
2. 使用Pop你就可以使任何`NSOject`子类中的任何property都实现动画，而不仅仅是那些在`UIKit`框架中被定义可以动画的那些类中的property。

Given that Pop has built-in support for decaying animations, implementing UIScrollView’s deceleration isn’t very tough.
考虑到Pop能提供很好的减速动画，所以要在我们自定义的ScrollView中实现UIScrollView里的减速动画是不难的。

The main idea is to animate the bounds’ origin in a decaying fashion using the velocity off the UIPanGestureRecognizer in its ended state.
主要想法是在UIPanGestureRecognizer的结束状态时，通过减小ScrollView的速率来实现减速效果。

So let’s begin. Start off by forking [my fork of Ole’s CustomScrollView project,](https://github.com/rounak/CustomScrollView) jump to the handlePanGesture: method in CustomScrollView.m. (Why start with my fork and not Ole’s project? He sets the translation to zero each time the gesture fires, which in turn resets the velocity. We, however, want the velocity to be preserved.)
让我们开始吧。首先我们先在GitHub上Fork[Ole的CustomScrollView项目](https://github.com/rounak/CustomScrollView)。跳到CustomScrollView.m中的`handlePanGesture: `方法。（为什么我们要重新fork项目，而不是直接用Ole 的项目？因为他在每次手势触发的时候，都将平移设置成为0，这样就会把速率重置为0，而速率正是我们的重要变量，所以我们需要保护它们。）

We want to start a decay animation only after the gesture has finished, so we’re interested in the UIGestureRecognizerStateEnded case of the switch statement.
我们想要在手势已经结束的时候才开始减速动画，所以我们只关心switch语句中`UIGestureRecognizerStateEnded`的情况。

Here’s the basic flow of the code fragment embedded below:
下面是代码的基本思路：

- We get the velocity of the gesture with the velocityInView: method.
- 我们通过`velocityInView: `方法得到手势的速率。

	CGPoint velocity = [panGestureRecognizer velocityInView:self];

- We do not want any movement in x or y direction if there’s not enough horizontal or vertical content respectively. So we add checks for that.
- 如果在水平方向或垂直方向没有足够空间那么我就不希望在x或y方向上能移动。所以我们要添加一个判断。

	if (self.bounds.size.width >= self.contentSize.width) {
		    velocity.x = 0;
	}
	if (self.bounds.size.height >= self.contentSize.height) {
		    velocity.y = 0;
	}

- Also it turns out that the velocity we get from the velocityInView: method is the negative of what we actually want, so fix that.
- 我们发现通过`velocityInView: `方法得到的速率实际上并不是我们想要的，所以我们要修复这个问题。

- We want to animate the bounds (specifically the bounds’ origin) so create a new POPDecayAnimation with the kPOPViewBounds property.
- 我们想要使界面的bounds（尤其是bound中的原点）动画，所以我们就创建了一个新的带kPOPViewBounds的POPDecayAnimation对象。

	POPDecayAnimation *decayAnimation = [POPDecayAnimation animationWithPropertyNamed:kPOPViewBounds];
 
- Pop expects the velocity to be of the same coordinate space as the property you’re trying to animate. The value will be an NSNumber for alpha (CGFloat), CGPoint wrapped in NSValue for center, CGRect wrapped in NSValue for bounds and so on. As you might have guessed, the change in the property value during the animation is a factor of the corresponding velocity component.
- Pop希望速率和你希望动画的property在相同的坐标系下。这个值可能是指向透明度的NSNumber（CGFloat），被封装在NSValue的CGPoint来指向界面的中心，被封装在NSValue的CGRect来指向bounds，等等。你可能已经猜到了，在动画过程中，property值的变化就相当于是速率。

- In our case, animating bounds means our velocity will be a CGRect, and its origin.x and origin.y values correspond to the change in the bounds' origin. Similarly, the size.width and size.height velocity components correspond to the change in the bounds’ size.
- 在我们这个项目中，我们要使bounds动画，这就意味我们的速率会是CGRect的变化引起的，并且它的原点的x和y值的变化相当于是bounds的原点上的变化。类似的，尺寸中的宽和高的速率就相当于bounds的尺寸变化。

- We don’t want to change the size of the bounds, so lets keep that velocity component zero always. Feed in the origin.x and origin.y values of the velocity with the pan gesture’s velocity. Wrap the whole thing into an NSValue and assign it to decayAnimation.velocity
- 我们不想改变bounds中的尺寸，所以就让尺寸的速率一直保持0。将原点的x和y值的速率作为滑动手势的速率输入。把这些值封装到一个NSValue中并将它赋值给decayAnimation.velocity。

	decayAnimation.velocity = [NSValue valueWithCGRect:CGRectMake(velocity.x, velocity.y, 0, 0)];

- Finally, add the decayAnimation to the view using the pop_addAnimation: method with any arbitrary key you want.
- 最后，通过使用`pop_addAnimation: `方法将decayAnimation添加到界面中，并且给界面添加任意想要的键值。

	[self pop_addAnimation:decayAnimation forKey:@"decelerate"];

Here’s the combined code:
这里是组合的代码：


	//get velocity from pan gesture
	//从滑动手势中得到速率
	CGPoint velocity = [panGestureRecognizer velocityInView:self];
	if (self.bounds.size.width >= self.contentSize.width) {
		    //make movement zero along x if no horizontal scrolling
		    //当水平方向没有空间滑动就把平移为0
		    velocity.x = 0; 
	}
	if (self.bounds.size.height >= self.contentSize.height) {
		    //make movement zero along y if no vertical scrolling
		    //当垂直方向没有空间滑动就把平移为0
		    velocity.y = 0; 
	}
	 
	//we need the negative velocity of what we get from the pan gesture, so flip the signs
	//我们需要将从滑动手势中得到的速率转换成负号，所以我们改变了一下符号。
	velocity.x = -velocity.x;
	velocity.y = -velocity.y;
	 
	POPDecayAnimation *decayAnimation = [POPDecayAnimation animationWithPropertyNamed:kPOPViewBounds];
	 
	//last two components zero as we do not want to change bound's size
	//最后两个零代表我们不想变化bound的尺寸
	decayAnimation.velocity = [NSValue valueWithCGRect:CGRectMake(velocity.x, velocity.y, 0, 0)];
	 
	[self pop_addAnimation:decayAnimation forKey:@"decelerate"];


![image](http://media.tumblr.com/185a108dfa8a706c90985b18198bd39c/tumblr_inline_n4z4hwnQiv1qh9cw7.gif)

With this you should have the basic deceleration working. You’ll notice that if your velocity is high, the view will scroll beyond its contentSize, but this could easily be solved by overriding setBounds: and adding checks to see that the bounds origin do not go beyond the thresholds. I’m surprised that Pop doesn’t add these checks itself.
通过上面的代码，我们已经实现了基本的减速动画。你可能会注意到，当你的速率很高的时候，这个界面会滑到超过他的`contentSize`，没关系，这个很容易解决，只要我们重写`setBounds: `方法并且添加判断这个bounds是否将要超过它contentSize就行了。但让我惊讶的是，Pop自己没有添加这些判断。

-----

A really interesting aspect of Pop is the ability to animate any property at all, not just ones declared as animatable by UIKit. Since we’re just animating the bounds’ origin, I thought this was a good chance to try out Pop`s custom property animation.
有意思的是POP能够让所有的property都能实现动画效果，而不仅仅是那些被UIKit定义可以实现动画的property。因为我们只是让bounds的原点动画了，所以我们可以实验一下POP是否真的能让所有的property都能动画。

Pop lets you do this is by giving you constant callbacks with the progress of the animation, so that you can update your view accordingly. It takes care of the hard part of handling timing, converting the timing into the value of your property, decaying or oscillating the animation etc.
POP能够这样做是因为它为我们提供了动画进度的常量回调函数，这样我们就能即使更新我们的界面。它实现了最难的部分--处理定时，将这些定时转换成你的property的值，减速或者震荡等等。

So here’s how you’ll do the same thing we did earlier, but now with your own custom property. Specifically, we’ll animate bounds.origin.x and bounds.origin.y. You’ll see that the structure of the code largely remains the same, except for the initialisation of this giant POPAnimatableProperty.

所以接下来就是做和最开始同样的事，但不同的是现在用自己自定义的property。这里我们想要bounds的原点的x和y动画。代码的结构大部分是差不多的，除了POPAnimatableProperty的初始化。

- The property name, as I understand, can be any arbitrary string. In this case, I’ve taken it to be @"com.rounak.bounds.origin" 
- property名字，个人理解，它可以是任意字符串。在这个项目中，我命名它为@"com.rounak.bounds.origin"

- There’s also an initializer block with POPMutableAnimatableProperty as the argument. (I think this initialisation pattern is called the builder pattern.)
- 这里也有一个使用POPMutableAnimatableProperty作为参数的初始化block。（我觉得这个初始化模式应该被命名为生成器模式）

- POPMutableAnimatableProperty has two important properties, readBlock and writeBlock. In readBlock you’ll have to feed data to Pop, and writeBlock you’ll have to retrieve that data, and update your view. The readBlock is called once whereas the writeBlock is called on each frame update of the display (via CADisplayLink).
- POPMutableAnimatableProperty有两个重要的property，readBlock 和 writeBlock。在readBlock中，你需要输入数据给POP，而在writeBlock中，你需要收回这些数据，然后更新你的界面。readBlcok只会被调用一次，而writeBlock会在每次更新显示（比如CADisplayLink）的帧时就会被调用。

- Pop internally converts everything into vectors, and hence it asks and gives you data in the form of a linear array of CGFloats.
- POP内部会将所有东西都转换成向量，因此他给我们提供的数据都是以由CGFloat组成的线性数组。

- In readBlock you simply assign bounds.origin.x and bounds.origin.y to values[0] and values[1] respectively.
- 在readBlock中我们简单地将bounds的原点x和y分别赋值给values[0]和values[1]。

	prop.readBlock = ^(id obj, CGFloat values[]) {
		    values[0] = [obj bounds].origin.x;
				    values[1] = [obj bounds].origin.y;
	};

- And in writeBlock you read values[0] (bounds.origin.x) and values[1] (bounds.origin.y) and update your view’s bounds.
- 然后在 writeBlock 中，你读取values[0](bounds.origin.x) 和 values[1](bounds.origin.y)，并且更新你界面中的bounds

	prop.writeBlock = ^(id obj, const CGFloat values[]) {
	    CGRect tempBounds = [obj bounds];
	    tempBounds.origin.x = values[0];
	    tempBounds.origin.y = values[1];
	    [obj setBounds:tempBounds];
	};

- The magic happens in the writeBlock which is called each time with values that follow a decay (or an oscillating) curve.
- 当writeBlock中的值随着减速（或震荡）曲线被每次调用时，神奇的事情发生了。

- You’ll notice that since we’re operating in just two dimensions here, our velocity is CGPoint, and not CGRect.
- 你会注意到当我们两个方向去操作时，我们的速率是CGPoint而不是CGRect。

Here’s the combined code:
下面是组合的代码：

	//get velocity from pan gesture
	//从滑动手势中得到速率
	CGPoint velocity = [panGestureRecognizer velocityInView:self];
	if (self.bounds.size.width >= self.contentSize.width) {
	    //make movement zero along x if no horizontal scrolling
	    //当没有水平方向滑动时就让x方向的移动为0
	    velocity.x = 0;
	}
	if (self.bounds.size.height >= self.contentSize.height) {
	    //make movement zero along y if no vertical scrolling
	    //当没有垂直方向滑动时就让y方向的移动为0
	    velocity.y = 0;
	}
	 
	//we need the negative velocity of what we get from the pan gesture, so flip the signs
	//我的需要从滑动手势中的到速率的负数，所以我们加上了符号。
	velocity.x = -velocity.x;
	velocity.y = -velocity.y;
	 
	POPDecayAnimation *decayAnimation = [POPDecayAnimation animation];
	 
	POPAnimatableProperty *prop = [POPAnimatableProperty propertyWithName:@"com.rounak.boundsY" initializer:^(POPMutableAnimatableProperty *prop) {
	    // read value, feed data to Pop
	    // 读取数据，输入数据给POP
	    prop.readBlock = ^(id obj, CGFloat values[]) {
	        values[0] = [obj bounds].origin.x;
	        values[1] = [obj bounds].origin.y;
	    };
	    // write value, get data from Pop, and apply it to the view
	    // 写数据，从POP中得到数据，并且将它应用到视图上。
	    prop.writeBlock = ^(id obj, const CGFloat values[]) {
	        CGRect tempBounds = [obj bounds];
	        tempBounds.origin.x = values[0];
	        tempBounds.origin.y = values[1];
	        [obj setBounds:tempBounds];
	    };
	    // dynamics threshold
	    动态临界值
	    prop.threshold = 0.01;
	}];
	 
	decayAnimation.property = prop;
	decayAnimation.velocity = [NSValue valueWithCGPoint:velocity];
	[self pop_addAnimation:decayAnimation forKey:@"decelerate"];

The entire project with the deceleration [can be found on GitHub](https://github.com/rounak/CustomScrollView/tree/custom-scroll-with-pop).
实现减速效果的完整项目[可以在GitHub中找到](https://github.com/rounak/CustomScrollView/tree/custom-scroll-with-pop)。