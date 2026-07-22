# Channel IDs & Category Filter Tokens

## Channel ID Enums (`channels` parameter)

Pass as comma-separated integer string (e.g. `channels=2,5`).

| ID | Channel |
| :--- | :--- |
| `1` | Web (news, blogs, websites) |
| `2` | Twitter/X |
| `3` | Facebook |
| `4` | YouTube |
| `5` | Instagram |
| `7` | Reviews (Google, TripAdvisor, Booking, etc.) |
| `99` | Other (LinkedIn, Reddit, TikTok, Pinterest, Telegram, Threads, Bluesky, Manual) |

## Category Filter Tokens (`categories` parameter)

Pass comma-separated tokens matching the channel definitions below.

### Web (channel 1)
`news`, `blog`, `website`, `social`, `reference`, `community`, `shop`, `comments`

### Twitter/X (channel 2)
`tweets`, `retweets`, `replies`, `quotes`, `twttag`

### Facebook (channel 3)
`fbpage`, `fbcomments`, `fbuser`, `fbshares`, `fbtag`, `fbad`, `fbstories`, `fbreels`

### YouTube (channel 4)
`youtubevideos`, `youtubecomments`

### Instagram (channel 5)
`instaposts`, `instacomments`, `instastories`, `instavideo`, `instatag`, `instareels`

### Reviews (channel 7)
`gplay` (Google Play), `gmb` (Google Business), `fbrv` (Facebook reviews), `bkng` (Booking), `trpadv` (TripAdvisor), `fs` (Foursquare), `htls` (Hotels), `trplt` (Trustpilot), `rvsio` (Reviews.io), `orv` (other reviews)

### Reddit (channel 99)
`rdposts`, `rdcomments`

### LinkedIn (channel 99)
`linkedin`, `linkedincomments`, `linkedinshares`

### TikTok (channel 99)
`tiktokposts`, `tiktokcomments`

### Telegram (channel 99)
`telegramposts`, `telegramcomments`

### Threads (channel 99)
`threadsposts`, `threadsreplies`

### Bluesky (channel 99)
`blueskyposts`, `blueskyreplies`, `blueskyquotes`

### Other (channel 99)
`pinterest`, `quora`, `disqus`, `manual`

## Usage Example

"Show me Instagram Reels and TikTok mentions" →

```
GET /v2/mentions?channels=5,99&categories=instareels,tiktokposts,tiktokcomments
```
