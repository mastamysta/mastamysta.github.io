# Object-Oriented vs. Data-Oriented Design

## Premise

We have a N bodies with random (double precision) mass floating in 2-dimensional space at random (discrete) positions. We need to calulate accellerations, velocities and new positions for each of the bodies over a thousand 1-second epochs, 
taking into account the effect of gravity between the objects. It is not considered acceptable to use an approximation method such as [Barnes-Hut] (https://en.wikipedia.org/wiki/Barnes%E2%80%93Hut_simulation), you must instead perform
an N^2 iteration for each epoch.

## Experiment Design

Just to ensure both methods actually calculate the same state, we'll use `rand()` from `<cmath>` to generate initial values for our bodies' mass, position, velocity etc. This will give use deterministic generation of initial values and so the outcome of the system should also be deterministic. We can then perform a checksum over the final state (e.g. sum all the velocities of all the bodies) to validate that both solutions got the same answer.

## Object Oriented Approach

I think that the intuitive object oriented design for this system is to create a class for a body which stores mass & velocity of the object. The code can be found [here](https://github.com/mastamysta/DoDDemo/blob/master/src/oo.cpp)

## Data-Oriented Approach

The data-oriented approach bundles logically similar *data* together rather than bundles of logically related objects. We end up with a `std::array` for velocities in the X direction, another for velocities in the Y direction and so on.

## Comparing the Two

Straight away we can validate that the two solutions are getting the same result, brilliant! We also end up with some interesting profiler data:

perf stat output for the object oriented approach:
```
 Performance counter stats for './src/oo':

          7,654.51 msec task-clock                       #    0.999 CPUs utilized             
               357      context-switches                 #   46.639 /sec                      
                 1      cpu-migrations                   #    0.131 /sec                      
               144      page-faults                      #   18.812 /sec                      
    28,533,505,080      cycles                           #    3.728 GHz                         (71.41%)
       472,115,946      stalled-cycles-frontend          #    1.65% frontend cycles idle        (71.42%)
    56,618,000,064      instructions                     #    1.98  insn per cycle            
                                                  #    0.01  stalled cycles per insn     (71.43%)
     6,923,760,016      branches                         #  904.534 M/sec                       (71.44%)
        49,877,247      branch-misses                    #    0.72% of all branches             (71.44%)
    25,479,432,948      L1-dcache-loads                  #    3.329 G/sec                       (71.44%)
        70,373,128      L1-dcache-load-misses            #    0.28% of all L1-dcache accesses   (71.42%)
   <not supported>      LLC-loads                                                             
   <not supported>      LLC-load-misses                                                       

       7.661313333 seconds time elapsed

       7.652126000 seconds user
       0.001999000 seconds sys
```

perf record output for object-oriented approach:
```
  34.19%  oo       libm.so.6             [.] __ieee754_pow_fma
  12.04%  oo       oo                    [.] double&& std::__pair_get<1ul>::__move_get<double, double>(std::p
  11.43%  oo       oo                    [.] space::calculate()
   9.19%  oo       libm.so.6             [.] pow@@GLIBC_2.29
   7.60%  oo       oo                    [.] calculate_gravitational_force(double, double, double)
   6.86%  oo       oo                    [.] std::tuple_element<1ul, std::pair<double, double> >::type&& std:
   5.45%  oo       oo                    [.] operator-(std::pair<double, double>, std::pair<double, double> c
   4.17%  oo       oo                    [.] double&& std::forward<double>(std::remove_reference<double>::typ
   2.39%  oo       oo                    [.] double&& std::__pair_get<0ul>::__move_get<double, double>(std::p
   2.07%  oo       oo                    [.] std::pair<double, double>::pair<double, double, true>(double&&,
   1.62%  oo       oo                    [.] std::remove_reference<std::pair<double, double>&>::type&& std::m
   1.12%  oo       oo                    [.] std::tuple_element<0ul, std::pair<double, double> >::type&& std:
   0.51%  oo       oo                    [.] pow@plt
   0.35%  oo       libm.so.6             [.] __pow_finite@GLIBC_2.15@plt
````

perf stat output for the data oriented approach:
```
 Performance counter stats for './src/dod':

          6,208.13 msec task-clock                       #    1.000 CPUs utilized             
                45      context-switches                 #    7.249 /sec                      
                 1      cpu-migrations                   #    0.161 /sec                      
               142      page-faults                      #   22.873 /sec                      
    21,910,062,076      cycles                           #    3.529 GHz                         (71.43%)
       493,341,230      stalled-cycles-frontend          #    2.25% frontend cycles idle        (71.43%)
    54,200,565,314      instructions                     #    2.47  insn per cycle            
                                                  #    0.01  stalled cycles per insn     (71.43%)
     6,023,458,545      branches                         #  970.253 M/sec                       (71.43%)
        51,512,393      branch-misses                    #    0.86% of all branches             (71.43%)
    22,371,890,766      L1-dcache-loads                  #    3.604 G/sec                       (71.43%)
        14,305,664      L1-dcache-load-misses            #    0.06% of all L1-dcache accesses   (71.43%)
   <not supported>      LLC-loads                                                             
   <not supported>      LLC-load-misses                                                       

       6.209944580 seconds time elapsed

       6.205894000 seconds user
       0.002999000 seconds sys
```

perf record output for data-oriented approach:
```
  54.47%  dod      libm.so.6             [.] __ieee754_pow_fma
  13.38%  dod      dod                   [.] space::calculate()
  12.16%  dod      dod                   [.] std::array<double, 1000ul>::operator[](unsigned long)
   8.87%  dod      libm.so.6             [.] pow@@GLIBC_2.29
   8.61%  dod      dod                   [.] calculate_gravitational_force(double, double, double)
   0.83%  dod      libm.so.6             [.] __pow_finite@GLIBC_2.15@plt
   0.58%  dod      dod                   [.] pow@plt
```

Interestingly, from the perspective of data-model optimisation it is a very good sign that we see a hotter hotspot on the data-oriented design. Designs with a poor data model tend to be slow *everywhere* whereas data-oriented designs tend to be very hot on one bounding section of code.

In this instance we can see that the `pow()` function is taking up a lot of time, because `pow()` for double precision numbers requires a whole bunch of expensive logarithm calculations!
