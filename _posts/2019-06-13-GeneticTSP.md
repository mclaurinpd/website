---
layout: post
title: "A Genetic Algorithm"
subtitle: "For the Traveling Salesman."
date: 2019-06-13 17:30:00 -0500
categories: code
description: Finding the best route.
---

###### I'm going to assume you already know what the Traveling Salesman Problem is. If you don't, don't fret! Go [here](https://en.wikipedia.org/wiki/Travelling_salesman_problem) to read about it and then come back!

You can easily come up with a potential solution for the TSP, but how can you check to make sure it's the best solution? Well... you have to look at all the solutions. Every single one. It's not so bad when there are only 3 or 4 cities. The number of possible solutions is **N!** where **N** is the number of cities. So there's only 6 possible solutions with 3 cities. 24 solutions when there are 4 cities. What about 14 cities? Well, that's eighty seven billion one hundred seventy eight million two hundred ninety one thousand two hundred possible solutions. Yikes.

This is going to be a series where I write code employing a certain kind of algorithm to find good solutions. You might think I would start with an exhaustive search, but I didn't. I started with a genetic algorithm instead.

A genetic algorithm employs some of the fundamentals of evolution to find a “good enough” solution. All you need for a genetic algorithm is a way to represent a solution of your problem genetically and a way to measure its fitness. In my case, a solution is represented as a sequence of cities. The fitness is measured as the total length of the trip starting with the length between the first and second cities and ending with the length between the last city and the first city. Once you've figured out how to do that you're ready to write the actual algorithm.

It's not too difficult to do it for the TSP. Generate a population of random solutions, breed the the good solutions with other good solutions and occasionally mutate some solutions to prevent convergence on a certain solution. Once you've generated new solutions, and replaced the old ones you start at the beginning by breeding the good ones and mutating. This is a very naive example. A more in-depth one might also use some other heuristics to prevent getting stuck at local minima and maxima. This might seem confusing but follow along with my code examples and it will make more sense!

I made a few classes, which you can look at [here.](https://github.com/mclaurinpd/TravelingSalesman/tree/master/TSPHelperClasses) You can also look in the [resources folder](https://github.com/mclaurinpd/TravelingSalesman/tree/master/TSP/resources) to see the 14 cities that I used.

The first one to look at is *City.cs* because it's the most simple class. It has three properties: a name, an x-coordinate, and a y-coordinate. The constructor sets these based on what is passed in. It also has an override of ToString() that returns the *Name* property. The second one I made is *Trip.cs* that is a class that contains a solution to the TSP. It has two properties. One is a list of *City.cs* and the other is the length of the trip. Most of the methods in it should be fairly self-explanatory but I will go over one that isn't. Take a look at *ShuffleRoute().* You'll see that it calls an extension method called *Shuffle().* If you didn't know, (.NET allows you to add methods to types that already exist.)[https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/extension-methods] So I did that and added the Fisher-Yates Shuffle as an extension.

```C#

using System;
using System.Collections.Generic;
using System.Text;

namespace Extensions
{
    static class ListExtensions
    {
        /// <summary>
        /// Shuffles a list with Fisher-Yates algorithm.
        /// </summary>
        /// <typeparam name="T"></typeparam>
        /// <param name="list"></param>
        /// <param name="rand"></param>
        public static void Shuffle<T>(this IList<T> list, Random rand)
        {
            int n = list.Count-1;
            while (n > 0)
            {
                n--;
                int k = rand.Next(n + 1);
                T value = list[k];
                list[k] = list[n];
                list[n] = value;
            }
        }
    }
}
```

It's fairly simple and a great way to make sure that all possible permutations are equally likely.

* Start at the end of the list, pick a random index between one and the length of the list.
* Swap the random index with the last element in the list.
* Repeat step one but decrease the upper limit of the random number by one.

This extension method is sued whenver a Trip object is instantiated. It assures that the route created is random!

The other method that needs a closer look is *Mutate(Random rand).*

```C#
        public void Mutate(Random rand)
        {
            int pos1 = rand.Next(Route.Count);
            int pos2 = rand.Next(Route.Count);
            while (pos1 == pos2)
            {
                pos2 = rand.Next(Route.Count);
            }
            var temp = Route[pos1];
            Route[pos1] = Route[pos2];
            Route[pos2] = temp;
        }
```

It's pretty clear what this method does. It swaps two random elements in the list. It *mutates* the list. Genetic diversity is important in real life and is also very important in a genetic algorithm. Let me explain why. Take a look at this table:

{:class="table table-bordered"}
| **Solution 1** | 1 | 1 | 0 | 0 | 1 |
|----------------|---|---|---|---|---|
| **Solution 2** | **0** | **1** | **1** | **0** | **1** |

No matter how many times we crossover(breed) these two arrays of bits, we'll never be able to have certain permuations. Notice how the second column of numbers has 1 in both solutions. How will we ever get a 0 in that position of the child solution that these two make? Eventually these two solutions will converge upon a certain solution and probably get stuck there. Unless we mutate. The function above does exactly that. Given a *mutation rate* it will decide whether or not to randomly swap two elements in the list. This is not the only way to mutate solutions for this algorithm, but it was the way that I chose.

Lastly, let's take a look at the actual [genetic algorithm.](https://github.com/mclaurinpd/TravelingSalesman/blob/master/GeneticTSP/GeneticAlgorithm.cs) The constructor takes in some parameters: population size, the number of generations, and the chance of mutation. All you have to do is create a new instance of the algorithm and it will run. I keep track of a few things and write them to a file for each generation. The best trip, the worst trip, the sample and population standard deviation, the mean, and the best overall solution. The fundamentals of the genetic algorithm are in here. Below is the crossover(breeding) method:

```C#
        public Trip Crossover(Trip a, Trip b)
        {
            var newRoute = new List<City>();
                        
            for (int i = 0; i < a.Route.Count / 2; i++)
            {
                newRoute.Add(a.Route[i]);
            }

            foreach (City city in b.Route)
            {
                if (!newRoute.Contains(city))
                {
                    newRoute.Add(city);
                }
            }

            var child = new Trip(newRoute);

            if (Rand.NextDouble() < MutateRate)
            {
                child.Mutate(Rand);
            }

            return child;
        }
```

There are many, many ways to change this around. I chose rather naively to take the first half of a random solution and add remaining cities of another random solution. This is naive for a few reason. It allows a solution to possibly crossover with itself. It also doesn't provide as much diversity as other methods. But it gets the job done quickly.

250 generations, a population of 2000, and a mutation rate of .2, I got these results:

{:class="table table-bordered"}
|                                | Generation 0 | Generation 250 |
|--------------------------------|--------------|----------------|
| Mean                           | 8.97 units   | 4.25 units     |
| Standard Deviation(Population) | .944         | .597           |
| Best Trip Length               | 5.52 units   | 3.94 units     |
| Worst Trip Length              | 11.70 units  | 7.13 units     |

It took it about 2.86 seconds to run. As you can see, the final generation had much more *fit* trips than the first generation. The best overall trip was not a part of the final generation though! The best trip that it found was in generation 75 with a length of 3.89 units. In another post I plan to do some generate a lot of data and try to figure out the best population size, generation number, and mutation rate. I hope you learned something from this and feel free to email me if you have questions.
