+++
title = 'How to Estimate Reading Time'
date = 2025-03-08T14:30:21+01:00
draft = false
readtime = true
toc = false
summary = "A short post to kick up this blog"
+++

We want to estimate how long it takes a reader to read a certain blog post.

If we assume that there exist a statistic such as the **average reading speed**
we can compute the reading time expressed in minutes by:

1. computing the number of words in the blog post
2. dividing (1) by the average reading speed (words-per-minute)

```python
def read_time_in_minutes(content: str) -> int:
  AVERAGE_READING_SPEED_WPM = ???
  return len(content.split(' ')) // AVERAGE_READING_SPEED_WPM
```

The problem now becomes assigning a value to `AVERAGE_READING_SPEED_WPM`.

Accordingly to a paper released in 2019 [[1]] the average reading speed for
adult English speakers is **238 WPM** for non-fiction work, and **183 WPM** for fiction work.

Most speakers fall in the 175-300 words-per-minute range,
with non-native speakers having a lower reading speed than native ones.

This blog post is built with the [Hugo](https://gohugo.io/) static-site generator
which includes the [ReadingTime](https://gohugo.io/methods/page/readingtime/) method
for computing the estimated reading time in minutes:

> By default, Hugo assumes a reading speed of 212 words per minute. For CJK languages, it assumes 500 words per minute.

The reading speed assumed by Hugo is not that far from the one suggested by the paper mentioned earlier.

Let's close up this post with the final version of `read_time_in_minutes`:

```python
def read_time_in_minutes(content: str) -> int:
  AVERAGE_READING_SPEED_WPM = 238
  return len(content.split(' ')) // AVERAGE_READING_SPEED_WPM
```

## References
1. Marc Brysbaert,
How many words do we read per minute? A review and meta-analysis of reading rate,
Journal of Memory and Language,
Volume 109,
2019,
104047,
ISSN 0749-596X,
https://doi.org/10.1016/j.jml.2019.104047.

[1]: https://www.sciencedirect.com/science/article/abs/pii/S0749596X19300786