# 35b coder bakeoff — aggregates (2026-07-26)

Valid runs: 120/120. Pass: 110. Spans assigned: 2203/2203.

## pass
| task | stock36 | stock35 | ornith | kat |
|---|---|---|---|---|
| yoke-bugfix | 5/5 | 4/5 | 5/5 | 5/5 |
| yoke-feature | 5/5 | 5/5 | 2/5 | 5/5 |
| log-parse | 4/5 | 5/5 | 5/5 | 4/5 |
| homelab-debug | 5/5 | 5/5 | 5/5 | 5/5 |
| web-research | 3/5 | 5/5 | 3/5 | 5/5 |
| mixed-pipeline | 5/5 | 5/5 | 5/5 | 5/5 |

## median_wall_s
| task | stock36 | stock35 | ornith | kat |
|---|---|---|---|---|
| yoke-bugfix | 90 | 77 | 105 | 97 |
| yoke-feature | 118 | 145 | 63 | 180 |
| log-parse | 71 | 110 | 61 | 101 |
| homelab-debug | 37 | 29 | 31 | 31 |
| web-research | 255 | 84 | 106 | 116 |
| mixed-pipeline | 103 | 68 | 68 | 66 |

## median_tools
| task | stock36 | stock35 | ornith | kat |
|---|---|---|---|---|
| yoke-bugfix | 8 | 6 | 11 | 10 |
| yoke-feature | 14 | 22 | 3 | 24 |
| log-parse | 23 | 34 | 18 | 32 |
| homelab-debug | 2 | 2 | 2 | 2 |
| web-research | 68 | 15 | 23 | 49 |
| mixed-pipeline | 13 | 16 | 14 | 12 |

## median_calls
| task | stock36 | stock35 | ornith | kat |
|---|---|---|---|---|
| yoke-bugfix | 12 | 12 | 16 | 11 |
| yoke-feature | 28 | 39 | 13 | 21 |
| log-parse | 15 | 43 | 16 | 11 |
| homelab-debug | 5 | 3 | 4 | 4 |
| web-research | 49 | 19 | 16 | 18 |
| mixed-pipeline | 15 | 18 | 18 | 10 |

## total_out_tokens
| task | stock36 | stock35 | ornith | kat |
|---|---|---|---|---|
| yoke-bugfix | 12718 | 10385 | 18720 | 10252 |
| yoke-feature | 42319 | 43012 | 54090 | 36061 |
| log-parse | 45970 | 37851 | 32327 | 27725 |
| homelab-debug | 12044 | 6729 | 9056 | 7827 |
| web-research | 122868 | 36648 | 34203 | 39597 |
| mixed-pipeline | 162133 | 71043 | 52771 | 31080 |

## total_think_chars
| task | stock36 | stock35 | ornith | kat |
|---|---|---|---|---|
| yoke-bugfix | 19530 | 17850 | 36696 | 9607 |
| yoke-feature | 51504 | 61007 | 89606 | 33363 |
| log-parse | 70390 | 47826 | 40977 | 19874 |
| homelab-debug | 14408 | 5904 | 9888 | 6612 |
| web-research | 219585 | 42079 | 46488 | 31270 |
| mixed-pipeline | 426767 | 72792 | 106207 | 37330 |

## total_in_tokens
| task | stock36 | stock35 | ornith | kat |
|---|---|---|---|---|
| yoke-bugfix | 1415844 | 1377739 | 1632946 | 1286817 |
| yoke-feature | 5069070 | 6668710 | 4117338 | 3589612 |
| log-parse | 3934035 | 6847739 | 2167279 | 2376442 |
| homelab-debug | 513297 | 285397 | 369498 | 287233 |
| web-research | 12211779 | 6465989 | 3757168 | 4045550 |
| mixed-pipeline | 1935339 | 3169712 | 1700617 | 970612 |

## per-arm totals (all 30 runs)
- stock36: 631 calls, in=25,079,364, out=398,052, think_chars=802,184
- stock35: 711 calls, in=24,815,286, out=205,668, think_chars=247,458
- ornith: 478 calls, in=13,744,846, out=201,167, think_chars=329,862
- kat: 383 calls, in=12,556,266, out=152,542, think_chars=138,056