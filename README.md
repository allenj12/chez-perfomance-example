# An example of Chez Scheme optimization

## Intro 
A while ago I decided to write a simple program to see how fast I could really make it in Chez Scheme. At the time I first started it, I was mainly interested in how memory layout would affect performance Chez. As time went on however I learned a few more things that I thought would be useful to write about.

In the process I wrote a simple "struct" macro that https://github.com/allenj12/struct that helps index proper data that you need over a bytevector. If you are familiar with the ftype interface in Chez, its similar except data is still managed by the garbage collector instead of being on the C heap. 

## The program.
The program is simple, it simulates a bunch of circles bounceing around on a screen. I did have a version that actually rendered the circles using SDL, but rendering became a bottleneck and made viewing performance difficult.

When a circle passes the edge of the "screen" it reverses the velocity for the respective access, and places the circle at the border. While it does this, it calculates how many times it processed the whole 500,000 circles in the last second, every second. 

The prorgam has this shape

```scheme
(import (chezscheme)
		(struct struct))

(define-type
	(struct circle
		(u16 x)
		(u16 y)
		(s8 vx)
		(s8 vy)))

(scheme-start
	 (lambda ()
	 	(define n-circles 500000)
	 	(define circles (circle-make n-circles))
		(define cap (fx* n-circles (type-sizeof circle)))
		(define radius 20)
		(define ticks (make-time 'time-duration 0 1))
		(define screen-width 1240)
		(define screen-height 800)
		
    ;;INITIALIZE CIRCLES WITH RANDOM VALUES
		(let init ([idx 0])
			(when (fx< idx cap)
				(circle-set! idx (x) circles (fx+ 200 (random 500)))
				(circle-set! idx (y) circles (fx+ 200 (random 500)))
				(circle-set! idx (vx) circles (fx- (random 30) 15))
				(circle-set! idx (vy) circles (fx- (random 20) 10))
				(init (fx+ idx (type-sizeof circle)))))

    ;;START INFININITE LOOP  WITH STARTING FPS
		(let inf ([t (current-time)]
				  [fps 0])
      ;;MAIN PROCESSING LOOP FOR THE CIRCLES
			(let loop ([idx 0])
				(when (fx< idx cap)
					(let* ([vx (circle-get idx (vx) circles)]
						  [vy (circle-get idx (vy) circles)]
						  [x (fx+ (circle-get idx (x) circles) vx)]
						  [y (fx+ (circle-get idx (y) circles) vy)])

              ;;IMPLEMENTATION HERE

        (loop (fx+ idx (type-sizeof circle))))))
;
				(if (time>? (time-difference (current-time) t) ticks)
					(begin (display "FPS: ") (display fps) (newline) (inf (current-time) 0))
					(inf t (fx1+ fps))))))
```

In the future code we will show will be where the implementation here is.


## Attempt 1

The reasoning here is we have 3 cases for each axis (6 total) where we check one side of the screen with the circle, if its not past bounds there then we check the other side, if its not past bounds there we assume the happy path and just increment x and y with the respective velocities.

```scheme
(cond
  ((fx< (fx- x radius) 0)
    (circle-set! idx (x) circles radius)
    (circle-set! idx (vx) circles (fx* -1 vx)))
  ((fx> (fx+ x radius) screen-width)
    (circle-set! idx (x) circles (fx- screen-width radius))
    (circle-set! idx (vx) circles (fx* -1 vx)))
  (else (circle-set! idx (x) circles x)))

(cond
  ((fx< (fx- y radius) 0)
    (circle-set! idx (y) circles radius)
    (circle-set! idx (vy) circles (fx* -1 vy)))
  ((fx> (fx+ y radius) screen-height)
    (circle-set! idx (y) circles (fx- screen-height radius))
    (circle-set! idx (vy) circles (fx* -1 vy)))
  (else (circle-set! idx (y) circles y)))
```

On my macbook air M3:

960 FPS

Can we do better?

## Attempt 2

Interestingly enough, although I originally set out to play around with memory layouts (and I did get some improvements that lead to struct library I wrote). At this point its all about managing the conditionals.

Improving here seems difficult, on paper this is minimum we have to check for each circle. However, the thing about this program is that hitting the bounds checks is rare, if we lift the normal path out of the conditional we see a huge improvement

```scheme
(circle-set! idx (x) circles x)
(circle-set! idx (y) circles y)

(cond
  ((fx< (fx- x radius) 0)
    (circle-set! idx (x) circles radius)
    (circle-set! idx (vx) circles (fx* -1 vx)))
  ((fx> (fx+ x radius) screen-width)
    (circle-set! idx (x) circles (fx- screen-width radius))
    (circle-set! idx (vx) circles (fx* -1 vx))))

(cond
  ((fx< (fx- y radius) 0)
    (circle-set! idx (y) circles radius)
    (circle-set! idx (vy) circles (fx* -1 vy)))
  ((fx> (fx+ y radius) screen-height)
    (circle-set! idx (y) circles (fx- screen-height radius))
    (circle-set! idx (vy) circles (fx* -1 vy))))
```

Here we get a pretty signifant boost: 

1100 FPS

## Attempt 3

So the game seems to be here to reduce the number of branches, even if it means writing to the bytevector multiple times etc...

Is there a way we can take these 2 remaining conditionals per axis, and turn them into one conditional each? The answer is yes because these are simple bounds checks, and a composition of min/max will do (fxmin/fxmax because we are using fixnums here).

```scheme
(circle-set! idx (x) circles x)
(circle-set! idx (y) circles y)

(when (or 
        (fx< x radius)
        (fx> x (fx- screen-width radius)))
  (circle-set! idx (x) circles
    (fxmin (fxmax radius x)
           (fx- screen-width radius)))
  (circle-set! idx (vx) circles (fx* -1 vx)))

(when (or 
        (fx< y radius)
        (fx> y (fx- screen-height radius)))
  (circle-set! idx (y) circles
    (fxmin (fxmax radius y)
           (fx- screen-height radius)))
  (circle-set! idx (vy) circles (fx* -1 vy)))
```

we now have one branch conditional for each axis. We are now at:

1430 FPS

## End
Thats as fast as I could push it, but with some simple changes we were able to get about a 50% improvement
