# `std::array` Is Slower Than `std::vector`?

This is a performance artifact that I came across while I was building out a pair of demo-programs to help motivate the practice of data-oriented-design. I was so intreagued that I decided to commit an entire post to revealing the cause of this particlar effect.

## The Setup

To understand this performance mystery, we need to go all the way back to the beginning. I was writing a test program which simulates the action of gravity between a whole bunch of randomly sized, randomly positioned masses. So far so good... here's the code:

shared.hpp:
```
#pragma once

#include <cmath>
#include <cstdlib>
#include <limits>

#define NUM_OBJECTS 1000
#define NUM_EPOCHS 1000
#define MAX_SPAWN_DISTANCE 100

#define G (6.67430e-11)

[[gnu::always_inline]]
auto calculate_gravitational_force(double m1, double m2, double distance) -> double
{
    if (distance == 0)
        return 0;

    return (G * m1 * m2) / (pow(distance, 2));
}

double generate_position()
{
    return rand() % MAX_SPAWN_DISTANCE;
}
```

oo.cpp
```
#include <vector>
#include <array>
#include <iostream>
#include <cstdint>

#include "shared.hpp"

static int next_id = 0;

class object
{
public:
    object()
    {
        id = next_id;
        next_id++;
        position = { generate_position(), generate_position() };
        velocity = { 0, 0 };
        mass = generate_position();
    }

    uint32_t id;
    std::pair<double, double> position, velocity;
    double mass;
};


std::pair<double, double> operator-(const std::pair<double, double> lhs,
                    const std::pair<double, double>& rhs)
{
    return std::pair(lhs.first - rhs.first, lhs.second - rhs.second);
}

auto operator+(const std::pair<double, double>& lhs,
                    const std::pair<double, double>& rhs)
{
    return std::pair(lhs.first + rhs.first, lhs.second + rhs.second);
}

auto operator*(const std::pair<double, double>& lhs, double rhs)
{
    return std::pair(lhs.first * rhs, lhs.second * rhs);
}

class space
{
public:
    space() : objects(std::vector<object>(NUM_OBJECTS, object())) {}

    auto calculate() -> void
    {
        for (auto& object: objects)
        {
            // Calculate force applied to object
            // Calculate accelleration, update velocity
            // Apply velocity
            auto force = std::pair<double, double>();
            auto& [fx, fy] = force;

            for (const auto& other: objects)
            {
                if (object.id == other.id)
                    continue;

                auto [dx, dy] = object.position - other.position;


                fx += calculate_gravitational_force(object.mass, other.mass, dx);
                fy += calculate_gravitational_force(object.mass, other.mass, dy);
            }

            auto accel = force * object.mass;

            // Apply accelleration for one unit-time:
            object.velocity = object.velocity + accel;
            
            // Apply velocity for one unit-time
            object.position = object.position + object.velocity;
        }
    }

    auto checksum() const -> double
    {
        double ret = 0;

        for (const auto& object: objects)
        {
            const auto& [px, py] = object.position;
            const auto& [vx, vy] = object.velocity;
            ret+= px + py + vx + vy;
        }

        return ret;
    }

private:
    std::vector<object> objects;
};

int main()
{
    space s;

    for (int i = 0; i < NUM_EPOCHS; i++)
        s.calculate();

    std::cout << s.checksum() << "\n";

    return 0;
}

```

...the code is straight-forward, but the key detail for this particular story is the fact that we store each of the massy bodies in a `std::vector`. I ran the program through perf, and got the following results:

```
 Performance counter stats for './src/oo':

         10,411.91 msec task-clock                       #    1.000 CPUs utilized             
                55      context-switches                 #    5.282 /sec                      
                 6      cpu-migrations                   #    0.576 /sec                      
               142      page-faults                      #   13.638 /sec                      
    25,799,276,342      cycles                           #    2.478 GHz                         (71.42%)
     3,051,154,007      stalled-cycles-frontend          #   11.83% frontend cycles idle        (71.42%)
    75,695,122,969      instructions                     #    2.93  insn per cycle            
                                                  #    0.04  stalled cycles per insn     (71.43%)
    12,098,369,447      branches                         #    1.162 G/sec                       (71.43%)
         1,451,827      branch-misses                    #    0.01% of all branches             (71.43%)
    46,489,290,312      L1-dcache-loads                  #    4.465 G/sec                       (71.44%)
       757,405,299      L1-dcache-load-misses            #    1.63% of all L1-dcache accesses   (71.42%)
   <not supported>      LLC-loads                                                             
   <not supported>      LLC-load-misses                                                       

      10.414508883 seconds time elapsed

      10.411061000 seconds user
       0.002001000 seconds sys
```

The high frontend stalls are explored in (the repository readme)[https://github.com/mastamysta/DoDDemo], but just to tidy up the code, I decided that `std::vector` was overkill and decided to try using `std::array`. The new code looks like this:

```
class space
{

    ...

private:
    std::array<object, NUM_OBJECTS> objects;
};
```

Bear in mind that this is the *only* change I made. When I reprofiled the program I got a really surprising result:

```

```
