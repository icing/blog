# curl Flame Graphs

Let's have some fun with flame graphs and curl!

I used curl's scorecard python script to add some `dtrace` of the curl process and [the nice FlameGraph](https://github.com/brendangregg/FlameGraph) by Brendan Gregg to have a look what curl is actually doing.

`dtrace` makes snapshots of the current stack frames here about 100 times per second and the Flame Graphs tools aggregate these and render an interactive SVG to look at the data. You can read these as follows:

* The overall width is the complete runtime of the curl process. But things are not really shown in the order they happen. While the left side *in general* started earlier than the more right entries, the are *aggregated*. When the same stack happens again later, they are accumulated in the first appearance.
* The width of a stack reflects the times this stack was seen. So, a frame of 20% width was observed in the process 20% of the overall runtime.
* The height reflects how deep the stack frames were nested.
* The colors are just for visual separation and have no further meaning. Lighter/Darker does **not** mean more CPU was used.

So, how does it look and what is there to see?

### Requests

Here, curl made 50000 requests to a localhost Apache, using HTTP/2 to retrieve a ~10KB resource.

![Curl Request Flame Graph](./images/curl.req.flames.svg)


