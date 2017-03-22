# JavaScript animations

JavaScript animations can handle things that CSS can't.

For instance, moving along a complex path, with a timing function different from Bezier curves, or an animation on a canvas.

[cut]

## setInterval

From the HTML/CSS point of view, an animation is a gradual change of the style property. For instance, changing `style.left` from `0px` to `100px` moves the element.

And if we increase it in `setInterval`, by making 50 small changes per second, then it looks smooth. That's the same principle as in the cinema: 24 or more frames per second is enough to make it look smooth.

The pseudo-code can look like this:

```js
let fps = 50; // 50 frames per second
let timer = setInterval(function() {
  if (animation complete) clearInterval(timer);
  else increase style.left
}, 1000 / fps)
```

The more complete example of the animation:
Более полный пример кода анимации:

```js
let start = Date.now(); // remember start time

let timer = setInterval(function() {
  // how much time passed from the start?
  let timePassed = Date.now() - start;

  if (timePassed >= 2000) {
    clearInterval(timer); // finish the animation after 2 seconds
    return;
  }

  // draw the animation at the moment timePassed
  draw(timePassed);

}, 20);

// as timePassed goes from 0 to 2000
// left gets values from 0px to 400px
function draw(timePassed) {
  train.style.left = timePassed / 5 + 'px';
}
```

Click for the demo:

[codetabs height=200 src="move"]

## requestAnimationFrame

Let's imagine we have several simultaneous animations.

If we run them separately, each one with its own `setInterval(..., 20)`, then the browser would have to repaint much more often than every `20ms`.

Each `setInterval` triggers once per `20ms`, but they are independent, so we have several independent runs within `20ms`.

These several independant actions should be grouped together, because it's easier for the browser to redraw things once per `20ms`.

In other words, this:

```js
setInterval(function() {
  animate1();
  animate2();
  animate3();
}, 20)
```

...Is lighter than this:

```js
setInterval(animate1, 20);
setInterval(animate2, 20);
setInterval(animate3, 20);
```

There's one more thing to keep in mind. Sometimes when CPU is overloaded or for other reasons it may be better to trigger redraws less often. Not 20, but maybe 200ms.

There's a standard [Animation timing](http://www.w3.org/TR/animation-timing/) that provides the function `requestAnimationFrame`.

It addresses all those issues and even more.

The syntax:
```js
let requestId = requestAnimationFrame(callback)
```

That schedules the `callback` function to run in the closest time when the browser wants to do animation.

If we do changes in elements in `callback` then they will be grouped together with other `requestAnimationFrame` callbacks and with CSS animations. So there will be one geometry recalculation and repaint instead of many.

The returned value `requestId` can be used to cancel the call:
```js
// cancel the scheduled execution of callback
cancelAnimationFrame(requestId);
```

The `callback` gets one argument -- the time passed from the beginning of the page load in microseconds. This time can also be obtained by calling [performance.now()](mdn:api/Performance/now).

Usually `callback` runs very soon, unless the CPU is overloaded or the laptop battery is almost discharged, or there's another reason.

The code below shows the time between first 20 runs fo `requestAnimationFrame`. Usually it's 10-20ms:

```html run height=40 refresh
<script>
  let prev = performance.now();
  let times = 0;

  requestAnimationFrame(function measure(time) {
    document.body.insertAdjacentHTML("beforeEnd", Math.floor(time - prev) + " ");
    prev = time;

    if (times++ < 10) requestAnimationFrame(measure);
  })
</script>
```

## Structured animation

Now we can make a more universal animation function based on `requestAnimationFrame`:

```js
function animate({timing, draw, duration}) {

  let start = performance.now();

  requestAnimationFrame(function animate(time) {
    // timeFraction goes from 0 to 1
    let timeFraction = (time - start) / duration;
    if (timeFraction > 1) timeFraction = 1;

    // calculate the current animation state
    let progress = timing(timeFraction)

    draw(progress); // draw it

    if (timeFraction < 1) {
      requestAnimationFrame(animate);
    }

  });
}
```

Function `animate` accepts 3 parameters that essentially describe the animation:

`duration`
: Total time of animation. Like, `1000`.

`timing(timeFraction)`
: Timing function, like CSS-property `transition-timing-function` that takes gets the fraction of time that passed (`0` at start, `1` at the end) and returns the animation completion (like `y` on the Bezier curve).

    For instance, a linear function means that the animation goes on uniformly with the same speed:

    ```js
    function linear(timeFraction) {
      return timeFraction;
    }
    ```

    It's graph:
    ![](linear.png)

    That's just like `transition-timing-function: linear`. There are more interesting variants shown below.

`draw(progress)`
: The function that takes the animation completion state and draws it. The value `progress=0` denotes the beginning animation state, and `progress=1` -- the end state.

    This is that function that actually draws out the animation.

    It can move the element:
    ```js
    function draw(progress) {
      train.style.left = progress + 'px';
    }
    ```

    ...Or do anything else, we can animate anything, in any way.


Let's animate the element `width` from `0` to `100%` using our function.

Click for the demo:

[codetabs height=60 src="width"]

The code for it:

```js
animate({
  duration: 1000,
  timing(timeFraction) {
    return timeFraction;
  },
  draw(progress) {
    elem.style.width = progress * 100 + '%';
  }
});
```

Unlike CSS animation, we can make any timing function and any drawing function here. The timing function is not limited by Bezier curves. And `draw` can go beyound properties, create new elements for like fireworks animation or something.

## Timing functions

We saw the simplest, linear timing function above.

Let's see more of them. We'll try movement animations with different timing functions to see how they work.

### Power of n

If we want to speed up the animation, we can use `progress` in the power `n`.

For instance, a parabolic curve:

```js
function quad(progress) {
  return Math.pow(progress, 2)
}
```

The graph:

![](quad.png)

See in action (click to activate):

[iframe height=40 src="quad" link]

...Or the cubic curve or event greater `n`. Increasing the power makes it speed up faster.

Here's the graph for `progress` in the power `5`:

![](quint.png)

In action:

[iframe height=40 src="quint" link]

### The arc

Function:

```js
function circ(timeFraction) {
  return 1 - Math.sin(Math.acos(timeFraction));
}
```

The graph:

![](circ.png)

[iframe height=40 src="circ" link]

### Back: bow shooting

This function does the "bow shooting". First we "pull the bowstring", and then "shoot".

Unlike previous functions, it depends on an additional parameter `x`, the "elasticity coefficient". The distance of "bowstring pulling" is defined by it.

The code:

```js
function back(x, timeFraction) {
  return Math.pow(timeFraction, 2) * ((x + 1) * timeFraction - x)
}
```

**The graph for `x = 1.5`:**

![](back.png)

For animation we use it with a concrete value of `x`. Example for `x = 1.5`:

[iframe height=40 src="back" link]

### Bounce

Imagine we are dropping a ball. It falls down, then bounces back a few times and stops.

The `bounce` function does the same, but in the reverse order: "bouncing" starts immediately. It uses few special coefficients for that:

```js
function bounce(timeFraction) {
  for (let a = 0, b = 1, result; 1; a += b, b /= 2) {
    if (timeFraction >= (7 - 4 * a) / 11) {
      return -Math.pow((11 - 6 * a - 11 * timeFraction) / 4, 2) + Math.pow(b, 2)
    }
  }
}
```

In action:

[iframe height=40 src="bounce" link]

### Elastic animation

One more "elastic" function that accepts an additional parameter `x` for the "initial range".

```js
function elastic(x, timeFraction) {
  return Math.pow(2, 10 * (timeFraction - 1)) * Math.cos(20 * Math.PI * x / 3 * timeFraction)
}
```

**The graph for `x=1.5`:**
![](elastic.png)

In action for `x=1.5`:

[iframe height=40 src="elastic" link]

## Reversal: ease*

So we have a collection of timing functions. Their direct application is called "easeIn".

Sometimes we need to show the animation in the reverse order. That's done with the "easeOut" transform.

### easeOut

In the "easeOut" mode the `timing` function is put into a wrapper `timingEaseOut`:

```js
timingEaseOut(timeFraction) = 1 - timing(1 - timeFraction)
```

In other words, we have a "transform" function `makeEaseOut` that takes a "regular" timing function and returns the wrapper around it:

```js
// accepts a timing function, returns the transformed variant
function makeEaseOut(timing) {
  return function(timeFraction) {
    return 1 - timing(1 - timeFraction);
  }
}
```

For instance, we can take the `bounce` function described above and apply it:

```js
let bounceEaseOut = makeEaseOut(bounce);
```

Then the bounce will be not in the beginning, but at the end of the animation. Looks even better:

[codetabs src="bounce-easeout"]

Here we can see how the transform changes the behavior of the function:

![](bounce-inout.png)

If there's an animation effect in the beginning, like bouncing -- it will be shown at the end.

In the graph above the <span style="color:#EE6B47">regular bounce</span> has the red color, and the <span style="color:#62C0DC">easeOut bounce</span> is blue.

- Regular bounce -- the object bounces at the bottom, then at the end sharply jumps to the top.
- After `easeOut` -- it first jumps to the top, then bounces there.

### easeInOut

We also can show the effect both in the beginning and the end of the animation. The transform is called "easeInOut".

Given the timing function, we calculate the animation state like this:

```js
if (timeFraction <= 0.5) { // first half of the animation
  return timing(2 * timeFraction) / 2;
} else { // second half of the animation
  return (2 - timing(2 * (1 - timeFraction))) / 2;
}
```

The wrapper code:

```js
function makeEaseInOut(timing) {
  return function(timeFraction) {
    if (timeFraction < .5)
      return timing(2 * timeFraction) / 2;
    else
      return (2 - timing(2 * (1 - timeFraction))) / 2;
  }
}

bounceEaseInOut = makeEaseInOut(bounce);
```

In action, `bounceEaseInOut`:

[codetabs src="bounce-easeinout"]

The "easeInOut" transform joins two graphs into one: `easeIn` (regular) for the first half of the animation and `easeOut` (reversed) -- for the second part.

The effect is clearly seen if we compare the graphs of `easeIn`, `easeOut` and `easeInOut` of the `circ` timing function:

![](circ-ease.png)

- <span style="color:#EE6B47">Red</span> is the regular variantof `circ` (`easeIn`).
- <span style="color:#8DB173">Green</span> -- `easeOut`.
- <span style="color:#62C0DC">Blue</span> -- `easeInOut`.

As we can see, the graph of the first half of the animation is the scaled down `easeIn`, and the second half is the scaled down `easeOut`. As a result, the animation starts and finishes with the same effect.

## More interesting "draw"

Instead of moving the element we can do something else. All we need is to write the write `draw`

### Typing in the text

Here's the animated "bouncing" text typing:

[codetabs src="text"]

## Summary

JavaScript animation is implemented with the help of `requestAnimationFrame`.

The helper `animate` function:

```js
function animate({timing, draw, duration}) {

  let start = performance.now();

  requestAnimationFrame(function animate(time) {
    // timeFraction goes from 0 to 1
    let timeFraction = (time - start) / duration;
    if (timeFraction > 1) timeFraction = 1;

    // calculate the current animation state
    let progress = timing(timeFraction);

    draw(progress); // draw it

    if (timeFraction < 1) {
      requestAnimationFrame(animate);
    }

  });
}
```

Options:

- `duration` -- the total animation time in ms.
- `timing` -- the function to calculate animation progress. Gets a time fraction from 0 to 1, returns the animation progress, usually from 0 to 1.
- `draw` -- the function to draw the animation.

Surely we could improve it, add more bells and whistles, but JavaScript animations are not applied on a daily basis. They are used to do something interesting and non-standard. So you'd want to add the features that you need when you need them.

JavaScript animations can use any timing function. We covered a lot of examples and transformations to make them even more versatile. Unlike CSS, we are not limited to Bezier curves here.

The same is about `draw`: we can animate anything, not just CSS properties.