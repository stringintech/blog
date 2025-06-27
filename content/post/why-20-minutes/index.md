---
title: "How Bitcoin's 10-Minute Block Interval Ends Up Being 20 Minutes"
date: 2025-04-12 00:00:00+0000
image:
tags:
  - Bitcoin
  - Blockchain
weight: 1
---


Earlier, I was reading an interesting [mathematical explanation](https://r6.ca/blog/20180225T160548Z.html) behind the paradox related to the interval between mined Bitcoin blocks. Despite the fact that the interval is 10 minutes on average, if you show up at a random time, you’ll see that the expected time until the next block appears is 10 minutes—regardless of how long you’ve already been waiting. Similarly, on average, you should expect that the previous block was mined 10 minutes ago. With this sampling approach, you end up with 10 minutes before and 10 minutes after, so overall, the interval between two consecutive blocks appears to be 20 minutes on average—not just 10 as you might initially think!

To explore what the article was saying, I had an idea. I thought: What if I generate lots of intervals whose average is 10 minutes, and then string them together to create a timeline? Next, I’d generate a bunch of random points along this timeline, and for each one, check which interval it falls into, calculating the distance from the point to the start of the interval and to the end. By averaging these distances, I should be able to see for myself that, on average, the end of the interval is 10 minutes away, the start of the interval block 10 minutes ago, and the full interval adds up to 20 minutes.

I wrote a little piece of code to do just that—and, unsuprisingly, the results matched my expectations:

```
Average block interval: 10.0339
Avg time to next block: 10.0594
Avg time since last block: 10.1122
Avg total interval length: 20.1717
```

```cpp
#include <iostream>
#include <vector>
#include <random>
#include <cmath>

int main()
{
    const int INTERVAL_COUNT = 100000;
    const int SAMPLE_COUNT = 100000;
    const double AVG_BLOCK_TIME = 10.0;

    // Setup random number generation
    std::random_device rd;
    std::mt19937 gen(rd());
    std::uniform_real_distribution<> uniform_dist(0.0, 1.0);

    // Generate exponentially distributed block intervals
    auto random_interval = [&]()
    {
        return -AVG_BLOCK_TIME * log(1.0 - uniform_dist(gen));
    };

    // Create a timeline of block boundaries
    std::vector<double> block_times{0};
    for (int i = 0; i < INTERVAL_COUNT; i++)
    {
        block_times.push_back(block_times.back() + random_interval());
    }

    // Calculate the actual average interval
    double timeline_end = block_times.back();
    double achieved_mean = timeline_end / INTERVAL_COUNT;
    std::cout << "Average block interval: " << achieved_mean << std::endl;

    // Sample random points on the timeline
    std::uniform_real_distribution<> timeline_dist(0.0, timeline_end);
    double time_to_next = 0.0;
    double time_since_last = 0.0;
    double total_interval = 0.0;

    for (int i = 0; i < SAMPLE_COUNT; i++)
    {
        // Pick a random point in time
        double point = timeline_dist(gen);

        // Find which interval contains this point
        auto it = std::upper_bound(block_times.begin(), block_times.end(), point) - 1;
        int idx = it - block_times.begin();

        // Calculate times to adjacent blocks
        double next_time = block_times[idx + 1] - point;
        double prev_time = point - block_times[idx];
        double interval_length = block_times[idx + 1] - block_times[idx];

        time_to_next += next_time;
        time_since_last += prev_time;
        total_interval += interval_length;
    }

    std::cout << "Avg time to next block: " << time_to_next / SAMPLE_COUNT << std::endl;
    std::cout << "Avg time since last block: " << time_since_last / SAMPLE_COUNT << std::endl;
    std::cout << "Avg total interval length: " << total_interval / SAMPLE_COUNT << std::endl;

    return 0;
}
```
