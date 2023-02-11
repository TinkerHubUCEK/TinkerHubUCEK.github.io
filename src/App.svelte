<script>
  import { onMount } from "svelte";
  import Logo from "./lib/Logo.svelte";
  let head;
  let elem = [];
  let colleges = ["UCEK", "TKME", "GECB", "LBSITW", "CET", "CECHENGANNUR"];
  import Typewriter from "typewriter-effect/dist/core";
  let screenCount = 0;
  onMount(() => {
    //hyperdrive
    const canvas = document.querySelector("#hyperdrive");
    const ctx = canvas.getContext("2d");

    // very simple mouse
    const mouse = { x: 0, y: 0, button: false };
    function mouseEvents(e) {
      mouse.x = e.pageX;
      mouse.y = e.pageY;
      mouse.button =
        e.type === "mousedown"
          ? true
          : e.type === "mouseup"
          ? false
          : mouse.button;
    }
    ["down", "up", "move"].forEach((name) =>
      document.addEventListener("mouse" + name, mouseEvents)
    );

    // High performance array pool using buubleArray to separate pool objects and active object.
    // This is designed to eliminate GC hits involved with particle systems and
    // objects that have short lifetimes but used often.
    // Warning this code is not well tested.
    const bubbleArray = () => {
      const items = [];
      var count = 0;
      return {
        clear() {
          // warning this dereferences all locally held references and can incur Big GC hit. Use it wisely.
          this.items.length = 0;
          count = 0;
        },
        update() {
          var head, tail;
          head = tail = 0;
          while (head < count) {
            if (items[head].update() === false) {
              head += 1;
            } else {
              if (tail < head) {
                const temp = items[head];
                items[head] = items[tail];
                items[tail] = temp;
              }
              head += 1;
              tail += 1;
            }
          }
          return (count = tail);
        },
        createCallFunction(name, earlyExit = false) {
          name = name.split(" ")[0];
          const keys = Object.keys(this);
          if (Object.keys(this).indexOf(name) > -1) {
            throw new Error(
              `Can not create function name '${name}' as it already exists.`
            );
          }
          if (!/\W/g.test(name)) {
            let func;
            if (earlyExit) {
              func = `var items = this.items; var count = this.getCount(); var i = 0;\nwhile(i < count){ if (items[i++].${name}() === true) { break } }`;
            } else {
              func = `var items = this.items; var count = this.getCount(); var i = 0;\nwhile(i < count){ items[i++].${name}() }`;
            }
            !this.items && (this.items = items);
            this[name] = new Function(func);
          } else {
            throw new Error(
              `Function name '${name}' contains illegal characters. Use alpha numeric characters.`
            );
          }
        },
        callEach(name) {
          var i = 0;
          while (i < count) {
            if (items[i++][name]() === true) {
              break;
            }
          }
        },
        each(cb) {
          var i = 0;
          while (i < count) {
            if (cb(items[i], i++) === true) {
              break;
            }
          }
        },
        next() {
          if (count < items.length) {
            return items[count++];
          }
        },
        add(item) {
          if (count === items.length) {
            items.push(item);
            count++;
          } else {
            items.push(items[count]);
            items[count++] = item;
          }
          return item;
        },
        getCount() {
          return count;
        },
      };
    };

    // Helpers rand float, randI random Int
    // doFor iterator
    // sCurve curve input -Infinity to Infinity out -1 to 1
    // randHSLA creates random colour
    // CImage, CImageCtx create image and image with context attached
    const randI = (min, max = min + (min = 0)) =>
      (Math.random() * (max - min) + min) | 0;
    const rand = (min = 1, max = min + (min = 0)) =>
      Math.random() * (max - min) + min;
    const doFor = (count, cb) => {
      var i = 0;
      while (i < count && cb(i++) !== true);
    }; // the ; after while loop is important don't remove

    const sCurve = (v, p) => 2 / (1 + Math.pow(p, -v)) - 1;
    const randHSLA = (
      h,
      h1,
      s = 100,
      s1 = 100,
      l = 50,
      l1 = 50,
      a = 1,
      a1 = 1
    ) => {
      return `hsla(${randI(h, h1) % 360},${randI(s, s1)}%,${randI(
        l,
        l1
      )}%,${rand(a, a1)})`;
    };
    const CImage = (w = 128, h = w) => (
      (c = document.createElement("canvas")), (c.width = w), (c.height = h), c
    );
    const CImageCtx = (w = 128, h = w) => (
      (c = CImage(w, h)), (c.ctx = c.getContext("2d")), c
    );

    // create image to hold text
    var textImage = CImageCtx(1024, 1024);
    var c = textImage.ctx;
    c.fillStyle = "#FF0";
    c.font = "64px arial black";
    c.textAlign = "center";
    c.textBaseline = "middle";
    const text =
      "HYPER,SPEED FX,VII,,Battle of Jank,,Hold the mouse,button to increase,speed.".split(
        ","
      );
    text.forEach((line, i) => {
      c.fillText(line, 512, i * 68 + 68);
    });
    const maxLines = text.length * 68 + 68;
    // canvas settings
    var w = canvas.width;
    var h = canvas.height;
    var cw = w / 2; // center
    var ch = h / 2;
    // diagonal distance used to set point alpha (see point update)
    var diag = Math.sqrt(w * w + h * h);
    // If window size is changed this is called to resize the canvas
    // It is not called via the resize event as that can fire to often and
    // debounce makes it feel sluggish so is called from main loop.
    function resizeCanvas() {
      points.clear();
      canvas.width = innerWidth;
      canvas.height = innerHeight;
      w = canvas.width;
      h = canvas.height;
      cw = w / 2; // center
      ch = h / 2;
      diag = Math.sqrt(w * w + h * h);
    }
    // create array of points
    const points = bubbleArray();
    // create optimised draw function itterator
    points.createCallFunction("draw", false);
    // spawns a new star
    function spawnPoint(pos) {
      var p = points.next();
      p = points.add(new Point());
      if (p === undefined) {
        p = points.add(new Point());
      }
      p.reset(pos);
    }
    // point object represents a single star
    function Point(pos) {
      // this function is duplicated as reset
      if (pos) {
        this.x = pos.x;
        this.y = pos.y;
        this.dead = false;
      } else {
        this.x = 0;
        this.y = 0;
        this.dead = true;
      }
      this.alpha = 0;
      var x = this.x - cw;
      var y = this.y - ch;
      this.dir = Math.atan2(y, x);
      this.distStart = Math.sqrt(x * x + y * y);
      this.speed = rand(0.01, 1);
      this.col = randHSLA(220, 280, 100, 100, 50, 100);
      this.dx = Math.cos(this.dir) * this.speed;
      this.dy = Math.sin(this.dir) * this.speed;
    }
    Point.prototype = {
      reset: Point, // resets the point
      update() {
        // moves point and returns false when outside
        this.speed *= hyperSpeed; // increase speed the more it has moved
        this.x += Math.cos(this.dir) * this.speed;
        this.y += Math.sin(this.dir) * this.speed;
        var x = this.x - cw;
        var y = this.y - ch;
        this.alpha =
          (Math.sqrt(x * x + y * y) - this.distStart) /
          (diag * 0.5 - this.distStart);
        if (
          this.alpha > 1 ||
          this.x < 0 ||
          this.y < 0 ||
          this.x > w ||
          this.h > h
        ) {
          this.dead = true;
        }
        return !this.dead;
      },
      draw() {
        // draws the point
        ctx.strokeStyle = this.col;
        ctx.globalAlpha = 0.25 + this.alpha * 0.75;
        ctx.beginPath();
        ctx.lineTo(
          this.x - this.dx * this.speed,
          this.y - this.dy * this.speed
        );
        ctx.lineTo(this.x, this.y);
        ctx.stroke();
      },
    };

    const maxStarCount = 400;
    const p = { x: 0, y: 0 };
    var hyperSpeed = 1.001;
    const alphaZero = sCurve(1, 2);
    var startTime;
    function loop(time) {
      if (startTime === undefined) {
        startTime = time;
      }
      if (w !== innerWidth || h !== innerHeight) {
        resizeCanvas();
      }
      // if mouse down then go to hyper speed
      if (mouse.button) {
        if (hyperSpeed < 1.75) {
          hyperSpeed += 0.01;
        }
      } else {
        if (hyperSpeed > 1.01) {
          hyperSpeed -= 0.01;
        } else if (hyperSpeed > 1.001) {
          hyperSpeed -= 0.001;
        }
      }
      // hyperspeed every 5 seconds
      if (time - startTime > 5000) {
        startTime = time;
        hyperSpeed = 1.75;
      }

      var hs = sCurve(hyperSpeed, 2);
      ctx.globalAlpha = 1;
      ctx.setTransform(1, 0, 0, 1, 0, 0); // reset transform

      //==============================================================
      // UPDATE the line below could be the problem. Remove it and try
      // what is under that
      //==============================================================
      //ctx.fillStyle = `rgba(0,0,0,${1-(hs-alphaZero)*2})`;

      // next two lines are the replacement
      ctx.fillStyle = "Black";
      ctx.globalAlpha = 1 - (hs - alphaZero) * 2;
      //==============================================================

      ctx.fillRect(0, 0, w, h);
      // the amount to expand canvas feedback
      var sx = (hyperSpeed - 1) * cw * 0.1;
      var sy = (hyperSpeed - 1) * ch * 0.1;

      // increase alpha as speed increases
      ctx.globalAlpha = (hs - alphaZero) * 2;
      ctx.globalCompositeOperation = "lighter";
      // draws feedback twice
      ctx.drawImage(canvas, -sx, -sy, w + sx * 2, h + sy * 2);
      ctx.drawImage(canvas, -sx / 2, -sy / 2, w + sx, h + sy);
      ctx.globalCompositeOperation = "source-over";

      // add stars if count < maxStarCount
      if (points.getCount() < maxStarCount) {
        var cent = (hyperSpeed - 1) * 0.5; // pulls stars to center as speed increases
        doFor(10, () => {
          p.x = rand(cw * cent, w - cw * cent); // random screen position
          p.y = rand(ch * cent, h - ch * cent);
          spawnPoint(p);
        });
      }
      // as speed increases make lines thicker
      ctx.lineWidth = 2 + hs * 2;
      ctx.lineCap = "round";
      points.update(); // update points
      points.draw(); // draw points
      ctx.globalAlpha = 1;

      // scroll the perspective star wars text FX
      var scrollTime = (time - startTime) / 5 - 2312;
      if (scrollTime < 1024) {
      }
      requestAnimationFrame(loop);
    }
    requestAnimationFrame(loop);

    //
    const typewriter = new Typewriter("h1", {
      loop: false,
      delay: 75,
      autoStart: true,
      strings: ["Web Development Reimagined"],
      pauseFor: 5000,
    });
    typewriter
      .typeString("A Collaboration of Six TinkerHubs")
      .callFunction(() => {
        head.style.transform = "translateY(-35vh)";
        head.style.transition = "transform 1s ease-in-out";
        elem.forEach((e, i) => {
          e.style.transitionDelay = `${i * 0.5}s`;
          e.style.opacity = 1;
        });
      })
      .pauseFor(5000)
      .callFunction(() => {
        head.style.transform = "translateY(0vh)";
        head.style.transition = "transform 1s ease-in-out";
        elem.forEach((e, i) => {
          e.style.transitionDelay = `0s`;
          e.style.opacity = 0;
        });
      })
      .deleteAll()
      .typeString("Coming Soon")
      .start();
  });
</script>

<div class="container">
  <div class="h1">
    <h1 bind:this={head}>Web Development Reimagined</h1>
    <div class="logo">
      {#each colleges as college, i}
        <div class="elem" bind:this={elem[i]}>
          <Logo name={college} />
        </div>
      {/each}
    </div>
  </div>
  <canvas id="hyperdrive" />
</div>

<style>
  @import url("https://fonts.googleapis.com/css2?family=Abel&display=swap");
  canvas {
    position: absolute;
    z-index: -1;
  }
  :global(body) {
    padding: 1rem;
    margin: 0;
    padding: 0;
    font-family: "Abel", sans-serif;
    width: 100vw;
    overflow-x: hidden;
    background: #000;
  }
  .container {
    text-align: center;
    min-height: 100vh;
    width: 100vw;
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    color: white;
    word-wrap: normal;
  }
  h1 {
    position: absolute;
    font-size: 8rem;
  }
  .h1 {
    height: 100vh;
    width: 100vw;
    overflow: hidden;
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    padding: 0;
    margin: 0;
  }
  h2 {
    font-size: 3rem;
  }
  h3 {
    font-size: 2rem;
  }
  .logo {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    grid-template-rows: repeat(2, 1fr);
    grid-gap: 1rem;
    justify-items: center;
    align-items: center;
    width: 80vw;
    height: 50vh;
    color: white;
    margin-top: 20rem;
  }
  .logo > .elem {
    width: 100%;
    height: 100%;
  }
  .elem {
    opacity: 0;
    transition: opacity 1s ease-in-out;
  }
  @media (max-width: 900px) {
    .logo {
      grid-template-columns: repeat(2, 1fr);
      grid-template-rows: repeat(3, 1fr);
    }
    h1 {
      font-size: 4rem;
    }
  }
</style>
