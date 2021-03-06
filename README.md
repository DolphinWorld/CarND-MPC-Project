# CarND-Controls-MPC
Self-Driving Car Engineer Nanodegree Program

--- 
## Project Writeup 


This model uses CppAD::ipopt to find the minimum cost provided by FG_eval operator method. There are six states (`psi`, `v`, `cte`, `epsi`, `delta`, and `a`), plus `x` and `y` positions. We then create cost function based on those six states. I mainly focus on cte and epsi, make sure it move to the correct reference trajectory. I also don't want the car to stop, so need to have cost function for v vs reference speed (`60 mph`). But the value of speed difference could be too big because their value are large, so I set the coefficiency of speed to 0.25. During test, I noticed big swing of the yellow line, I think that may because my angle and delta angle is too large, so I increased the cost of both coefficient to large values, 1200 and 1000. 

### Choices for `dt` and `N`

Since I want to calculate the cte as soon as possible, I set `dt` to `0.1`. Also, since we only use one or two points of predicted path, I set `N` to small number `10`. I tried to set the `dt` to `0.2`, the accuracy dropped, the car drive toward edge twice, so I just keep it `0.1` to be a safe self-driving car. I tried to set `N` to some bigger number, but realized it is not useful because the car only took the first one or two data points, and it took longer time to calculate. The N * dt is the prediction path length. 

### A polynomial is fitted to waypoints.

From json message, I got two vectors for `ptsx` and `ptsy`, and an angle `psi`. But `psi` is from plane coordinates to car's per I used the following code to convert them to car's coordinates.

```c
    double cos_psi = cos(-psi);
    double sin_psi = sin(-psi);
    for (int i = 0; i < N; i++) {
        double dx = ptsx[i] - px;
        double dy = ptsy[i] - py;

        x_path(i) = dx * cos_psi - dy * sin_psi;
        y_path(i) = dx * sin_psi + dy * cos_psi;
    }
```

---

### Update 

I used the following statement to update the cte, epsi:

```c
    auto coeffs = polyfit(x_path, y_path, 3);
    double cte = polyeval(coeffs, 0); 
    double epsi = -atan(coeffs[1]);

    px = v * 1.67 / 3600 * lag; // cos(psi) with small value is almost 1
    py = 0;  // the sin(psi) with small psi is almost 0
    v  = v + prev_a_ * lag;
```

---

### Model details.

There are two actuators, delta and a, which is tuned by `IPOPT` to make sure the cost are low. In my cost function, I set both values and their diffs with previous values to the cost function, to make the car move and turn smoothly. 

The actuators value set to fields of `steering_angle` and `throttle` in the json output. The steering angle divided by `deg2rad(25)`, and turned to negative, because it is positive number when turn right, while normal angle is positive when turn anti-clockwise. I also set the `mpc_x` and `mpc_y` with the predicted position information.

I use the return value indexed `delta_start` and `a_start` from `IPOPT::solve_result" as `steering_angle` and `throttle` to update in the json message. The value of steering_angle will be divided by `deg2rad(25)`.


--- 

### Lag compensation 

I use default `100 ms` latency specified in main.cpp. I adjusted my equation to do some kind of prediction like following:

```c
	double lag = dt / 3600; // dt is 0.1
        px = v * cos(psi) * lag;
        py = v * sin(psi) * lag;
        v  = v + prev_a_ * lag;
        cte = cte + (v * sin(epsi) * lag);
        epsi = epsi + v * prev_delta_ / Lf * lag;
```


--- 

## Dependencies

* cmake >= 3.5
 * All OSes: [click here for installation instructions](https://cmake.org/install/)
* make >= 4.1
  * Linux: make is installed by default on most Linux distros
  * Mac: [install Xcode command line tools to get make](https://developer.apple.com/xcode/features/)
  * Windows: [Click here for installation instructions](http://gnuwin32.sourceforge.net/packages/make.htm)
* gcc/g++ >= 5.4
  * Linux: gcc / g++ is installed by default on most Linux distros
  * Mac: same deal as make - [install Xcode command line tools]((https://developer.apple.com/xcode/features/)
  * Windows: recommend using [MinGW](http://www.mingw.org/)
* [uWebSockets](https://github.com/uWebSockets/uWebSockets)
  * Run either `install-mac.sh` or `install-ubuntu.sh`.
  * If you install from source, checkout to commit `e94b6e1`, i.e.
    ```
    git clone https://github.com/uWebSockets/uWebSockets 
    cd uWebSockets
    git checkout e94b6e1
    ```
    Some function signatures have changed in v0.14.x. See [this PR](https://github.com/udacity/CarND-MPC-Project/pull/3) for more details.
* Fortran Compiler
  * Mac: `brew install gcc` (might not be required)
  * Linux: `sudo apt-get install gfortran`. Additionall you have also have to install gcc and g++, `sudo apt-get install gcc g++`. Look in [this Dockerfile](https://github.com/udacity/CarND-MPC-Quizzes/blob/master/Dockerfile) for more info.
* [Ipopt](https://projects.coin-or.org/Ipopt)
  * Mac: `brew install ipopt`
       +  Some Mac users have experienced the following error:
       ```
       Listening to port 4567
       Connected!!!
       mpc(4561,0x7ffff1eed3c0) malloc: *** error for object 0x7f911e007600: incorrect checksum for freed object
       - object was probably modified after being freed.
       *** set a breakpoint in malloc_error_break to debug
       ```
       This error has been resolved by updrading ipopt with
       ```brew upgrade ipopt --with-openblas```
       per this [forum post](https://discussions.udacity.com/t/incorrect-checksum-for-freed-object/313433/19).
  * Linux
    * You will need a version of Ipopt 3.12.1 or higher. The version available through `apt-get` is 3.11.x. If you can get that version to work great but if not there's a script `install_ipopt.sh` that will install Ipopt. You just need to download the source from the Ipopt [releases page](https://www.coin-or.org/download/source/Ipopt/).
    * Then call `install_ipopt.sh` with the source directory as the first argument, ex: `sudo bash install_ipopt.sh Ipopt-3.12.1`. 
  * Windows: TODO. If you can use the Linux subsystem and follow the Linux instructions.
* [CppAD](https://www.coin-or.org/CppAD/)
  * Mac: `brew install cppad`
  * Linux `sudo apt-get install cppad` or equivalent.
  * Windows: TODO. If you can use the Linux subsystem and follow the Linux instructions.
* [Eigen](http://eigen.tuxfamily.org/index.php?title=Main_Page). This is already part of the repo so you shouldn't have to worry about it.
* Simulator. You can download these from the [releases tab](https://github.com/udacity/self-driving-car-sim/releases).
* Not a dependency but read the [DATA.md](./DATA.md) for a description of the data sent back from the simulator.


## Basic Build Instructions


1. Clone this repo.
2. Make a build directory: `mkdir build && cd build`
3. Compile: `cmake .. && make`
4. Run it: `./mpc`.

## Tips

1. It's recommended to test the MPC on basic examples to see if your implementation behaves as desired. One possible example
is the vehicle starting offset of a straight line (reference). If the MPC implementation is correct, after some number of timesteps
(not too many) it should find and track the reference line.
2. The `lake_track_waypoints.csv` file has the waypoints of the lake track. You could use this to fit polynomials and points and see of how well your model tracks curve. NOTE: This file might be not completely in sync with the simulator so your solution should NOT depend on it.
3. For visualization this C++ [matplotlib wrapper](https://github.com/lava/matplotlib-cpp) could be helpful.

## Editor Settings

We've purposefully kept editor configuration files out of this repo in order to
keep it as simple and environment agnostic as possible. However, we recommend
using the following settings:

* indent using spaces
* set tab width to 2 spaces (keeps the matrices in source code aligned)

## Code Style

Please (do your best to) stick to [Google's C++ style guide](https://google.github.io/styleguide/cppguide.html).

## Project Instructions and Rubric

Note: regardless of the changes you make, your project must be buildable using
cmake and make!

More information is only accessible by people who are already enrolled in Term 2
of CarND. If you are enrolled, see [the project page](https://classroom.udacity.com/nanodegrees/nd013/parts/40f38239-66b6-46ec-ae68-03afd8a601c8/modules/f1820894-8322-4bb3-81aa-b26b3c6dcbaf/lessons/b1ff3be0-c904-438e-aad3-2b5379f0e0c3/concepts/1a2255a0-e23c-44cf-8d41-39b8a3c8264a)
for instructions and the project rubric.

## Hints!

* You don't have to follow this directory structure, but if you do, your work
  will span all of the .cpp files here. Keep an eye out for TODOs.

## Call for IDE Profiles Pull Requests

Help your fellow students!

We decided to create Makefiles with cmake to keep this project as platform
agnostic as possible. Similarly, we omitted IDE profiles in order to we ensure
that students don't feel pressured to use one IDE or another.

However! I'd love to help people get up and running with their IDEs of choice.
If you've created a profile for an IDE that you think other students would
appreciate, we'd love to have you add the requisite profile files and
instructions to ide_profiles/. For example if you wanted to add a VS Code
profile, you'd add:

* /ide_profiles/vscode/.vscode
* /ide_profiles/vscode/README.md

The README should explain what the profile does, how to take advantage of it,
and how to install it.

Frankly, I've never been involved in a project with multiple IDE profiles
before. I believe the best way to handle this would be to keep them out of the
repo root to avoid clutter. My expectation is that most profiles will include
instructions to copy files to a new location to get picked up by the IDE, but
that's just a guess.

One last note here: regardless of the IDE used, every submitted project must
still be compilable with cmake and make./
