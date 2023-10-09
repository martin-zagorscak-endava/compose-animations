<h1>Compose animations standard</h1>

<h2>Table of contents</h2>

<!-- TOC -->
  * [Motivation](#motivation)
  * [Compose animations cheat sheet](#compose-animations-cheat-sheet)
  * [NavHost transitions between screens](#navhost-transitions-between-screens)
  * [Item placement animations (lists and grids)](#item-placement-animations-lists-and-grids)
  * [Motion layout](#motion-layout)
  * [Shared elements transition](#shared-elements-transition)
  * [Guidelines](#guidelines)
<!-- TOC -->

## Motivation

## Compose animations cheat sheet

![compose_animation_cheat_sheet.png](images%2Fcompose_cheat_sheet.png)

**More on: https://developer.android.com/jetpack/compose/animation/introduction**

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
                animationSpec = tween(durationMillis = 700)
            )
        },
        exitTransition = {
            slideOutOfContainer(
                towards = AnimatedContentTransitionScope.SlideDirection.Left,
                animationSpec = tween(700)
            )
        },
        popEnterTransition = {
            slideIntoContainer(
                towards = AnimatedContentTransitionScope.SlideDirection.Right,
                animationSpec = tween(700)
            )
        },
        popExitTransition = {
            slideOutOfContainer(
                towards = AnimatedContentTransitionScope.SlideDirection.Right,
                animationSpec = tween(700)
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
        ) { Screen("Screen A", backgroundColor = Color.Red) }
        composable(route = Screens.ScreenB.route) { Screen("Screen B", backgroundColor = Color.Yellow) }
        composable(route = Screens.ScreenC.route) { Screen("Screen C", backgroundColor = Color.Green) }
    }
```

## Item placement animations (lists and grids)

## Motion layout

## Shared elements transition

![shared-elements-transition-demo.gif](images%2Fshared-elements-transition-demo.gif)
![shared-elements-transition-demo1.gif](images%2Fshared-elements-transition-demo1.gif)

`TODO: Shared element transitions are currently in development`

For now, you can use [this](https://github.com/mxalbert1996/compose-shared-elements) library.

## Guidelines

https://m2.material.io/design/motion/understanding-motion.html#principles
https://m3.material.io/styles/motion/overview
https://uxdesign.cc/the-ultimate-guide-to-proper-use-of-animation-in-ux-10bd98614fa9
