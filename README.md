<h1>Compose animations standard</h1>

<h2>Table of contents</h2>

<!-- TOC -->

* [Motivation](#motivation)
* [Compose animations cheat sheet](#compose-animations-cheat-sheet)
* [NavHost transitions between screens](#navhost-transitions-between-screens)
* [MotionLayout](#motionlayout)
* [Shared elements transition](#shared-elements-transition)
* [Guidelines](#guidelines)
    * [Duration](#duration)
    * [Easing](#easing)
    * [What makes a good transition?](#what-makes-a-good-transition)
    * [Transition patterns](#transition-patterns)
        * [Container transform](#container-transform)
        * [Forward and backward](#forward-and-backward)
        * [Lateral](#lateral)
        * [Top level](#top-level)
        * [Enter and exit](#enter-and-exit)
        * [Skeleton loaders](#skeleton-loaders)

<!-- TOC -->

## Motivation

Animations can make your app more **engaging**, **interactive**, and **visually appealing**. They can also help to
improve the user experience by providing feedback and making transitions between screens smoother. Jetpack Compose makes
it easy to implement the animations into the app with just a few lines of code, even if you are new to animations.

This document covers basic common animations (shown in the cheat sheet) and also a little more complex cases which
are common or could appear in our projects. Hopefully, with its simplicity, this document will engage you to implement
animations in your project.

## Compose animations cheat sheet

![compose_animation_cheat_sheet.png](images%2Fcompose_cheat_sheet.png)

**More on: https://developer.android.com/jetpack/compose/animation/introduction**

## Compose animation decision tree

Don't know what animation API to use for your animation implementation? Here with this decision tree you can choose the
corresponding API. 

![compose_animation_decision_tree.jpeg](images%2Fcompose_animation_decision_tree.jpeg)

## NavHost transitions between screens

To set transitions for all navigation destinations inside `NavHost` define `enterTransition` and `exitTransition`. By
default `NavHost` uses `fadeIn(animationSpec = tween(700))` for enter transition
and `fadeOut(animationSpec = tween(700))` for exit transition. Optionally, you can also set the `popEnterTransition`
and `popExitTransition`, by default these values are same as `enterTransition` and `exitTransition`. On the other hand,
if you want to set transition for a specific screen, you need to define transitions at the `composable` function which
is placed inside `NavGraphBuilder` scope.

For enter transitions you can choose between 4 categories:

- fade: `fadeIn`
- scale: `scaleIn`
- slide: `slideIn`, `slideInHorizontally`, `slideInVertically`
- expand: `expandIn`, `expandHorizontally`, `expandVertically`

For exit transitions you can also choose between the same 4 categories which represent the opposite transition effects:

- fade: `fadeOut`
- scale: `scaleOut`
- slide: `slideOut`, `slideOutHorizontally`, `slideOutVertically`
- shrink: `shrinkOut`, `shrinkHorizontally`, `shrinkVertically`

For combining enter transitions you can simply add them up with `+` operator, thanks to:

``` kotlin
sealed class EnterTransition {
    // ...
    operator fun plus(enter: EnterTransition): EnterTransition {
        return EnterTransitionImpl(
            TransitionData(
                fade = data.fade ?: enter.data.fade,
                slide = data.slide ?: enter.data.slide,
                changeSize = data.changeSize ?: enter.data.changeSize,
                scale = data.scale ?: enter.data.scale
            )
        )
    }
    // ...
}
```

For example: `fadeIn() + scaleIn()`. The order of the transitions being combined does not matter, as these
transitions will start simultaneously. The order of applying transforms from these enter transitions (if defined)
is: alpha and scale first, shrink or expand, then slide. The same applies to exit transitions.

``` kotlin
NavHost(
    navController = navController,
    startDestination = Screens.ScreenA.route,
    // Transitions are applied for all defined routes for its NavGraph
    enterTransition = {
        slideIntoContainer(
            towards = AnimatedContentTransitionScope.SlideDirection.Left,
            animationSpec = tween(durationMillis = 500)
        )
    },
    exitTransition = {
        slideOutOfContainer(
            towards = AnimatedContentTransitionScope.SlideDirection.Left,
            animationSpec = tween(durationMillis = 500)
        )
    },
    popEnterTransition = {
        slideIntoContainer(
            towards = AnimatedContentTransitionScope.SlideDirection.Right,
            animationSpec = tween(durationMillis = 500)
        )
    },
    popExitTransition = {
        slideOutOfContainer(
            towards = AnimatedContentTransitionScope.SlideDirection.Right,
            animationSpec = tween(durationMillis = 500)
        )
    }
) {
    // NavGraphBuilder scope
    composable(
        route = Screens.ScreenA.route,
        // These transitions are applied only for this route
        enterTransition = { fadeIn() },
        exitTransition = { fadeOut() }
        // If not explicity defined, pop transitions are the same as enter and exit transition
    ) { Screen("Screen A") }
    composable(route = Screens.ScreenB.route) { Screen("Screen B") }
    composable(route = Screens.ScreenC.route) { Screen("Screen C") }
}
```

## MotionLayout

**MotionLayout** allows you to animate the motion of views between different states, creating dynamic and engaging
experiences for the user. MotionLayout is built on top of Jetpack Composes ConstraintLayout, which gives you a flexible
and expressive way to lay out your views. MotionLayout adds to this by allowing you to define transitions between
different ConstraintLayout states. These transitions can be triggered by user interaction, such as a button click or
scroll gesture.

Before getting into the implementation of MotionLayout in Compose, lets explain some common MotionLayout terminology:

- **MotionLayout** - A MotionLayout API for the old View framework
- **MotionCompose** - A MotionLayout API for Jetpack Compose
- **MotionScene** - A file that defines the various `ConstraintSets`, `Transitions` and `Keyframes` for a MotionLayout
  animation
- **ConstraintSet** - A set of constraints that define the initial and final layout states, along with any intermediate
  states, for a MotionLayout
- **Transition** - The animation sequence that occurs between two or more Constraint Sets in a MotionLayout
- **Keyframe** - Allows modifying points between the transitions by frames
- **KeyAttribute** - A property of a view that can be animated during a MotionLayout transition, such as its position,
  size or alpha value

Implementation steps will be explained for a simple example in which button click will trigger the movement of a circle
from start of the screen to the end of it:

![motion_layout_example.gif](images%2Fmotion_layout_example.gif)

Firstly, to work with MotionLayout in Jetpack Compose, dependency for constraint layout for Compose is needed:
`implementation("androidx.constraintlayout:constraintlayout-compose:$versionCode")`. After that you can start composing
your apps UI with the `MotionLayout` function, to which you need to pass `motionScene` (set of constraint rules which UI
elements will follow through transitions), `progress` (float type indicator - value which variates between `0f`
and `1f`) and `content` (composables which will transition from one state to another). To create a motion scene and
define the constraints, create a `.json` file placed as `res/raw/motion_scene.json5`. To start writing the necessary
constraints we need to
follow [this pattern](https://github.com/androidx/constraintlayout/wiki/Compose-MotionLayout-JSON-Syntax):

```
{
  Header:{}
  Design:{}
  Variables:{}
  ConstraintSets:{}
  Transitions: {
       // Transition Named default (special name)
      default:{
          KeyFrames:{
              KeyPositions: {}
              KeyAttributes: {}
              KeyCycles: {}
          }
      }
  }
}
```

For this simple example, only the `ConstraintSets` and `Transitions` properties will be defined.

After creating the motion scene .json file (which is empty for now), implement the `MotionLayout` function inside your
screen composable, for this simple example it will look like this:

``` kotlin
@Composable
fun Screen(modifier: Modifier = Modifier) {
    val context = LocalContext.current
    // loading the motion scene file
    val motionScene = remember {
        context.resources
            .openRawResource(R.raw.motion_scene_screen_b)
            .readBytes()
            .decodeToString()
    }
    
    var isClicked by rememberSaveable { mutableStateOf(false) }
    // progress smoothly changes by changing the isClicked state ie. clicking on the button
    val progress by animateFloatAsState(targetValue = if (isClicked) 1f else 0f, animationSpec = tween(2000))
    
    MotionLayout(
        motionScene = MotionScene(content = motionScene),
        progress = progress,
        modifier = modifier.fillMaxSize()
    ) {
        Box(
            modifier = Modifier
                .size(48.dp)
                .clip(CircleShape)
                .background(Color.Red)
                // set the layoutId so you can reach it in motion_scene.json5
                .layoutId("circle")
        )
        OutlinedButton(
            // changes the state of the motion layout
            onClick = { isClicked = !isClicked },
            colors = ButtonDefaults.outlinedButtonColors(containerColor = Color.White, contentColor = Color.Red),
            border = BorderStroke(2.dp, Color.Red),
            modifier = Modifier.layoutId("button")
        ) {
            Text(text = "Move the Circle")
        }
    }
}
```

**Note: MotionLayout in ConstraintLayout-Compose 1.0.0 only supports JSON syntax while in ConstraintLayout-Compose 1.1.0
supports DSL & JSON syntax. There are different signatures of the `MotionLayout` function, so you can also write the
motion scene constraint sets in Kotlin, setting the MotionScene parameters `start` and `end`. But, the best practice
would be separate the motion scene in another file as mentioned earlier, because the content can easily grow and it may
become harder to read and to maintain.**

Now that we set up the composable part, lets define the constraints for the `circle` and the `button` as we defined the
layout ids in Compose:

```
{
  ConstraintSets:{
    start: {
      // layoutIds we defined in the composables modifiers
      circle: {
        top: ['parent', 'top', 0],
        bottom: ['parent', 'bottom', 0],
        start: ['parent', 'start', 8],
        // custom properties which can be named arbitrarily
        custom: {
          background_color: "#FF0000"
        }
      },
      button: {
        top: ['circle', 'bottom', 36],
        start: ['parent', 'start', 0],
        end: ['parent', 'end', 0]
      }
    },
    end: {
      circle: {
        top: ['parent', 'top', 0],
        bottom: ['parent', 'bottom', 0],
        end: ['parent', 'end', 8],
        custom: {
          background_color: "#0000FF"
        }
      },
      button: {
        top: ['circle', 'bottom', 36],
        start: ['parent', 'start', 0],
        end: ['parent', 'end', 0]
      }
    }
  }
}
```

As you have seen at the code snippet above, you can define custom properties which will differ for the start and the
final state. To use `background_color` custom property in Compose, reach for it inside `MotionLayoutScope` (block
where the composables are defined) like this:

``` kotlin
val circleProperties = customProperties("circle")
val circleBackgroundColor = circleProperties.color("background_color")
```

And replace the hardcoded colors with `circleBackgroundColor` (defined in the motion scene file) where you want to apply
it.

**Note: Beside colors, you can also define numerical values such as `Int`s, `Float`s, font sizes and distances in `Dp`s
as a custom property.**

The last thing to do is to define the transition inside motion scene file:

```
{
  ConstraintSets: { // ... },
  Transitions: {
    default: {
      from: 'start',
      to: 'end',
      // used for modifying points between the transitions
      KeyFrames: {
        KeyAttributes: [
          {
            // the target array can hold mutiple ids
            target: ['circle'],
            // frames in percentages
            frames: [0, 50, 100],
            // for each frame there needs to be a value for the wanted property
            // these values are applied while transitioning from one transition state to another
            translationY: [0, -100, 0],
            custom: {
              background_color: ["#FF0000", "#00FF00", "#0000FF"]
            }
          }
        ],
      }
    }
  }
}
```

For more details of ConstraintLayout and MotionLayout, [click here](https://github.com/androidx/constraintlayout/wiki).

## Shared elements transition

![shared-elements-transition-demo.gif](images%2Fshared-elements-transition-demo.gif)
![shared-elements-transition-demo1.gif](images%2Fshared-elements-transition-demo1.gif)

`TODO: Shared element transitions are currently in development`

For now, you can use [this](https://github.com/mxalbert1996/compose-shared-elements) library.

## Guidelines

These guidelines will guide you how to apply animations appropriately in terms of duration (how long a transition
lasts), easing (acceleration over time) and transformation (how UI element takes a different form). Also, for common use
cases there are some transition patterns which can be useful while building your interactive app.

### Duration

When elements change their state or position, the duration of the animation should be slow enough to give users the
possibility to notice the change, but at the same time quick enough not to cause waiting. If the animation or transition
speed is not appropriate it will result with poor UX, which counters your idea to improve the existing app. To achieve
better UX, choosing the right combination of duration and easing will produce with smooth and responsive transitions.

![durations.gif](images%2Fdurations.gif)

[Numerous researches](https://valhead.com/2016/05/05/how-fast-should-your-ui-animations-be/) have discovered that
optimal speed for interface animation is between 200 and 500 ms. These figures
are based on the particular qualities of the human brain. Any animation shorter than 100 ms is instantaneous and won’t
be recognized at all. Whereas the animation longer than 1 second would convey a sense of delay and thus be boring for
the user.

On the mobile devices, [Material Design Guidelines](https://m2.material.io/design/motion/speed.html#duration) also
suggests limiting the duration of animation to 200–300 ms. As for tablets, the duration should be longer by 30% — around
400–450 ms. The reason is simple: the size of the screen is bigger so objects overcome the longer path when they change
position. On wearables, the duration should be accordingly 30% shorter — around 150–200 ms, because on a smaller screen
the distance to travel is shorter.

**Do:** A transition with a well tuned duration is quick and easy to follow <br>
![example_smooth_transition.gif](images%2Fexample_smooth_transition.gif)

**Don't:** A transition with a well tuned duration is quick and easy to follow <br>
![example_too_fast_transition.gif](images%2Fexample_too_fast_transition.gif)

**Durations are chosen based on these criteria:**

**Transition size**

Transitions that cover small areas of the screen have short durations, while large areas have long durations. Scaling
duration with the size of a transition gives a consistent sense of speed.

This transition covers a small area with a short 200ms duration:<br>
![example_small_area_transition.gif](images%2Fexample_small_area_transition.gif)

This transition covers a large area with a short 500ms duration:<br>
![example_large_area_transition.gif](images%2Fexample_large_area_transition.gif)

**Enter vs exit transitions**

**Transitions that exit**, dismiss or collapse an element **use shorter durations**. Exit transitions are faster because
they require less attention than the user's next task.

**Transitions that enter** or remain persistent on the screen **use longer durations**. This helps users focus attention
on what's new on the screen.

![example_enter_exit_transition.gif](images%2Fexample_enter_exit_transition.gif)

**1)** An Enter transition has a long duration of 500ms<br>
**2)** An Exit transition has a short duration of 200ms

### Easing

Easing helps to make the movement of the object more natural. It’s one of the essential principles of the animation. For
the animation not to look mechanical and artificial, the object should move with some acceleration or deceleration —
just like all live objects in the physical world. So it is suggested to avoid linear motion and use the more natural
ones.

Animation with easing looks more natural compared to the linear one<br>
![easing_vs_no_easing.gif](images%2Feasing_vs_no_easing.gif)

There are different types of motion of UI elements, but the most common ones are:

- **Linear motion or constant speed motion**
  ![linear_motion.gif](images%2Flinear_motion.gif)
- **Ease-in or acceleration motion**
  ![ease_in_motion.gif](images%2Fease_in_motion.gif)
- **Ease-out or deceleration motion**
  ![ease_out_motion.gif](images%2Fease_out_motion.gif)
- **Ease-in-out or standard motion**
  ![ease_in_out_motion.gif](images%2Fease_in_out_motion.gif)

Real mobile examples when using ease motions:

- **Acceleration curve for throwing the card out of the screen**
  ![ease_in_example.gif](images%2Fease_in_example.gif)
- **Deceleration curve for a nice show-up**
  ![ease_out_example.gif](images%2Fease_out_example.gif)
- **The navigation drawer hides from the screen with the standard curve**
  ![ease_in_out_example.gif](images%2Fease_in_out_example.gif)

### What makes a good transition?

A well-designed transition should follow these rules:

- **Have an accessibility setting** which helps users with a sensitivity to motion. When that setting is on, transitions
  should **use subtle fades instead of intense sliding or scaling animations** and **disable decorative effects like
  parallax or shape morphing**.
- Transition should be **consistent**, applying the right type of transition helps make apps feel cohesive and
  predictable to use.
- Use **stable layouts** to avoid unnecessary distractions. Skeleton loaders are a great solution, it
  keeps UI elements coherent and stable during a transition. Avoid content shifting positions or instantly popping in as
  it loads. It can be distracting and frustrating to use.
- **Have a unified direction of movement**. Elements are grouped and move along a primary axis instead of moving in
  independent directions. Only important elements like hero images remain persistent throughout the transition.
- **Fully fade out content before fading new content in**. This avoids the overlap of partially transparent elements
  resulting in distracting and messy frames. If a cross-fade needs to occur, keep it quick and hide it during the
  fastest part of the transition.
- **Don't slowly fade components on top of other content as they enter or exit**. This creates distracting cross-faded
  frames. If a fade is needed, like with a Dialog entering the middle of the screen, the fade should use a short
  duration to hide that part of the transition.
- **Keep the style of the motion simple**. Transitions are not receptive to highly stylized motion. They're frequent,
  often occupy large portions of the screen, and are primarily meant to help users accomplish a task.

**Jump cuts should generally be avoided** as a default setting since they can be disorienting. Instantly transitioning
from one screen to the next offers no clues to help a user orient themselves. If pure efficiency is a top priority, like
opening a menu in a productivity app, a jump cut may be preferred.

### Transition patterns

Transitions are short animations that connect individual elements or full-screen views of an app. They are fundamental
to a great user experience because they help users understand how an app works. Well-designed transitions make an
experience feel high quality and expressive. They should be the top priority for a strong motion implementation.

#### Container transform

This pattern is used to seamlessly transform an element to show more detail, like a card expanding into a details page.

Commonly used with: **Cards**, **lists**, **image galleries**, **search boxes**, **sheets**, **FABs** and **chips**

Persistent elements are used to seamlessly connect the start and end state of the transition. The most common persistent
element is a container, which is a shape used to represent an enclosed area. It can also be an important element, like a
hero image. Of all transition patterns, this one creates the strongest relationship between elements. It's also
perceived to be the most expressive. This pattern is highly effective at creating a relationship between elements. It's
also the most dramatic pattern in terms of style and should be reserved for the right context. Consider using it for:

- Hero moments that should be expressive
- Shallow hierarchies where you expand an element for more detail then collapse it
- Creating a seamless connection between elements

**Note:** Don't use container transform in apps with deep hierarchies, the motion becomes excessive. The expressive
style also doesn't fit this utility focused navigation.

![container_transform.gif](images%2Fcontainer_transform.gif)

#### Forward and backward

This pattern is used for navigating between screens at consecutive levels of hierarchy, like navigating from an inbox to
a message thread.

Commonly used with: **Lists**, **cards**, **buttons**, **links**

A horizontal sliding motion indicates moving forward or backward between screens. Android uses a fade as screens slide.
This reduces the amount of motion, since the screens don't have to slide the full width of the device.

![forward_and_backward_transition.gif](images%2Fforward_and_backward_transition.gif)

#### Lateral

This pattern is used for navigating between peer content at the same level of hierarchy, like swiping between tabs of a
content library.

Commonly used with: **Tabs**, **carousels**, and **image galleries**

Lateral transitions use a sliding motion similar to a forward and backward pattern, but it does not use a fade or
parallax effect. Instead elements are grouped and slide in unison, creating a strong peer relationship. This also hints
at being able to gesturally swipe elements to navigate.

**Note:** Don't use a Lateral transition for navigating hierarchical screens. Sliding content the full width of the
screen is excessive for a high frequency transition. It also implies an equal peer relationship which isn't accurate to
the hierarchy of the screens.

![lateral_transition.gif](images%2Flateral_transition.gif)

#### Top level

This pattern is used to navigate between top-level destinations of an app, like tapping a destination in a Navigation
bar.

Commonly used with: **Navigation bar**, **navigation rail**, and **navigation drawer**

The exiting screen quickly fades out and then the entering screen fades in. Since the content of top level destinations
isn't necessarily related, the motion intentionally does not use grouping or persistent elements to create a strong
relationship between screens.

**Note:** Don't use a lateral transition to move between top level destinations. It implies you can swipe between top
level destinations which can conflict with other components like carousels or swipe-able list items.

![top_level_transition.gif](images%2Ftop_level_transition.gif)

#### Enter and exit

This pattern is used to introduce or remove a component on the screen. Components can enter and exit **within the screen
bounds**, like a dialog appearing over an app. They can also enter and exit by **crossing the screen bounds**, like a
navigation drawer or bottom sheet that slides on and off screen.

**Within screen bounds**

Android components expand and collapse along the x or y axis as they enter and exit. Scale and z-axis motion is avoided
since they imply elevation change, which doesn't match M3's reduced elevation model.

Commonly used with: **FABs**, **dialogs**, **menus**, **snackbars**, **time pickers** and **tooltips**

![enter_and_exit_within_screen_bounds.gif](images%2Fenter_and_exit_within_screen_bounds.gif)

**Beyond screen bounds**

Android components expand and collapse along the x or y axis as they slide on and off screen. This emphasizes their
shape, making an otherwise simple transition more expressive.

Commonly used with: **App bars**, **banners**, **navigation bar**, **navigation rail**, **navigation drawer** and
**sheets**

**Note:** Don't use this pattern for navigating hierarchical screens. Sliding content the full height of the screen is
excessive, and it creates an unclear relationship between screens.

![enter_and_exit_beyond_screen_bounds.gif](images%2Fenter_and_exit_beyond_screen_bounds.gif)

#### Skeleton loaders

This pattern is used to transition from a temporary loading state to a fully loaded UI. Skeleton loaders are UI
abstractions that hint at where content will appear once it's loaded. They're used in combination with other transitions
to reduce perceived latency and stabilize layouts as content loads. They have a subtle pulsing animation to
indicate indeterminate progress. Once content is loaded, it quickly fades in on top of the skeleton loader.

![skeleton_transition.gif](images%2Fskeleton_transition.gif)

**For more details about transition patterns
check [this link](https://m3.material.io/styles/motion/transitions/transition-patterns).**
