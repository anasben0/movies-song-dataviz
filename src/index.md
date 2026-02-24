---
title: Movie Song Trends
---

# Movie & Soundtrack Trends Dashboard

This dashboard was created as a group project exploring the relationship between
movies and their soundtracks through Google Trends data. The dataset contains
100 movies and series shown in France, each paired with a notable song from its
soundtrack, with monthly search interest scores ranging from 0 to 100.

---

## Graph 1 — How does the popularity lifecycle of a cinematic release compare to that of its official soundtrack over time?
```js
const raw_dataset = await FileAttachment("data/movie-song-trends_dataset.json").json();
```
```js
const movieTitles = [...new Set(raw_dataset.map(d => d.movie_data.title))];
const selectedMovieTitle = view(Inputs.select(movieTitles, { label: "Select a movie" }));
```
```js
const selectedEntry = raw_dataset.find(d => d.movie_data.title === selectedMovieTitle);

const rawMovieScores = selectedEntry.trends.map(t => t.scores.movie === "<1" ? 0.5 : +t.scores.movie);
const rawSongScores  = selectedEntry.trends.map(t => t.scores.song  === "<1" ? 0.5 : +t.scores.song);

const maxMovie = Math.max(...rawMovieScores);
const maxSong  = Math.max(...rawSongScores);

const formattedData = selectedEntry.trends.map((t, i) => ({
  date:             new Date(t.date + "-01"),
  movieRelativePop: maxMovie > 0 ? (rawMovieScores[i] / maxMovie) * 100 : 0,
  songRelativePop:  maxSong  > 0 ? (rawSongScores[i]  / maxSong)  * 100 : 0,
  rawMovieScore:    rawMovieScores[i],
  rawSongScore:     rawSongScores[i]
}));
```
```js
Plot.plot({
  title: `Normalized Comparative Analysis: ${selectedMovieTitle}`,
  subtitle: "Movie vs. Soundtrack Peak Intensity (0-100% Scale)",
  width: width,
  height: 500,
  marginLeft: 50,
  y: {
    label: "Relative Popularity (%)",
    domain: [0, 100],
    grid: true
  },
  x: {
    label: "Timeline",
    grid: true
  },
  marks: [
    Plot.differenceY(formattedData, {
      x: "date",
      y1: "movieRelativePop",
      y2: "songRelativePop",
      positiveFill: "rgba(44, 160, 44, 0.4)",
      negativeFill: "rgba(255, 127, 14, 0.4)"
    }),
    Plot.lineY(formattedData, { x: "date", y: "movieRelativePop", stroke: "#2ca02c", strokeWidth: 2 }),
    Plot.lineY(formattedData, { x: "date", y: "songRelativePop",  stroke: "#ff7f0e", strokeWidth: 2 }),
    Plot.ruleX(formattedData, Plot.pointerX({
      x: "date",
      stroke: "gray",
      strokeWidth: 1,
      strokeDasharray: "4,4",
      channels: {
        "Movie Rel. Pop. (%)": d => Math.round(d.movieRelativePop),
        "Raw Movie Score":     "rawMovieScore",
        "Song Rel. Pop. (%)":  d => Math.round(d.songRelativePop),
        "Raw Song Score":      "rawSongScore"
      },
      tip: true
    })),
    Plot.ruleY([0])
  ]
})
```

---

## Graph 2 — Does recent movies use more "trendy" songs?

### How this chart was built

**Processing the data.** The raw data is processed to extract meaningful metrics.
To strictly eliminate the bias of a movie boosting its own soundtrack popularity,
a 12-month time window strictly before each movie's release date is computed
dynamically. The monthly search trends are then filtered to isolate this exact
period and the average song popularity is computed. Unquantifiable string values
like "<1" are parsed as 0.5 to allow mathematical operations without crashing
the calculation.

**Calculating and attributing the tiers.** Instead of using arbitrary hardcoded
thresholds, exact quantiles at 20%, 40%, 60%, and 80% are calculated based on
the distribution of movie popularity scores across the entire dataset. This
ensures movies are divided into five statistically balanced segments: Obscure,
Niche, Moderate, Mainstream, and Blockbuster. Movies released before 2015 are
filtered out to ensure the graph strictly answers the research question regarding
recent trends.

**Creating the graph.** Observable Plot builds an interactive scatterplot with a
custom color scale for the five tiers. To manage outliers and prevent extreme
values from squashing the scale, a visual cap is set at a Y-axis limit of 20.
Normal scores are plotted as standard dots, while outliers above 20 are mapped
as triangles pinned to the top of the axis, with interactive tooltips revealing
their true values. Two distinct linear regressions are displayed: a dashed gray
line representing the global baseline for all data, and a solid black line that
recalculates dynamically based on the tiers selected via the interactive
checkboxes.
```js
const allTiers = ["Blockbuster", "Mainstream", "Moderate", "Niche", "Obscure"];
const selectedTiers = view(Inputs.checkbox(allTiers, { value: allTiers, label: "Filter by tier" }));
```
```js
const Y_CAP = 20;

const processedData = raw_dataset.map(d => {
  const releaseDate = new Date(d.movie_data.release_date);
  const oneYearBefore = new Date(releaseDate);
  oneYearBefore.setFullYear(oneYearBefore.getFullYear() - 1);

  const yearPriorTrends = d.trends.filter(t => {
    const trendDate = new Date(t.date + "-01");
    return trendDate >= oneYearBefore && trendDate < releaseDate;
  });

  const avgSongPop = yearPriorTrends.reduce((sum, t) =>
    sum + (t.scores.song === "<1" ? 0.5 : parseFloat(t.scores.song)), 0
  ) / (yearPriorTrends.length || 1);

  return {
    title:        d.movie_data.title,
    full_date:    releaseDate,
    movie_pop:    parseFloat(d.movie_data.popularity),
    song_pop_avg: avgSongPop
  };
});

const pops = processedData.map(d => d.movie_pop).sort((a, b) => a - b);
const getQuantile = p => pops[Math.floor(p * (pops.length - 1))];
const thresholds = {
  obs:  getQuantile(0.2),
  nic:  getQuantile(0.4),
  mod:  getQuantile(0.6),
  main: getQuantile(0.8)
};

const finalData = processedData
  .filter(d => d.full_date.getFullYear() >= 2015)
  .map(d => {
    let tier = "Blockbuster";
    if      (d.movie_pop <= thresholds.obs)  tier = "Obscure";
    else if (d.movie_pop <= thresholds.nic)  tier = "Niche";
    else if (d.movie_pop <= thresholds.mod)  tier = "Moderate";
    else if (d.movie_pop <= thresholds.main) tier = "Mainstream";
    return { ...d, tier };
  });

const tierColors = {
  "Blockbuster": "#2d5a27",
  "Mainstream":  "#7cb342",
  "Moderate":    "#fbc02d",
  "Niche":       "#fb8c00",
  "Obscure":     "#e53935"
};
```
```js
Plot.plot({
  grid: true,
  height: 500,
  x: { label: "Release date →", domain: [new Date("2015-01-01"), new Date("2026-01-01")] },
  y: { label: "↑ Song popularity (avg. 1 year before release)", domain: [0, Y_CAP] },
  color: { domain: Object.keys(tierColors), range: Object.values(tierColors), legend: true },
  marks: [
    Plot.linearRegressionY(finalData, {
      x: "full_date", y: "song_pop_avg",
      stroke: "#ccc", strokeDasharray: "4,4", ci: null
    }),
    Plot.dot(finalData.filter(d => selectedTiers.includes(d.tier) && d.song_pop_avg <= Y_CAP), {
      x: "full_date", y: "song_pop_avg", fill: "tier", tip: true,
      title: d => `${d.title}\nPop: ${d.song_pop_avg.toFixed(2)}`
    }),
    Plot.dot(finalData.filter(d => selectedTiers.includes(d.tier) && d.song_pop_avg > Y_CAP), {
      x: "full_date", y: Y_CAP, fill: "tier", symbol: "triangle", r: 6, tip: true,
      title: d => `${d.title}\nOUTLIER: ${d.song_pop_avg.toFixed(2)}`
    }),
    Plot.linearRegressionY(finalData.filter(d => selectedTiers.includes(d.tier)), {
      x: "full_date", y: "song_pop_avg",
      stroke: "black", strokeWidth: 2, ci: null
    }),
    Plot.text(["Selected trend"], { x: new Date("2025-06-01"), y: Y_CAP - 1, textAnchor: "end" }),
    Plot.text(["Overall trend"],  { x: new Date("2025-06-01"), y: 1,         textAnchor: "end", fill: "#aaa" })
  ]
})
```

---

## Graph 3 — Do American songs get more traction than French songs in movies?

### How this chart was built

The dataset is processed to extract the peak Google Trends score for each song —
the highest monthly score recorded over the entire available period. Google
Trends sometimes returns the string "<1" instead of 0 when a song has negligible
but non-zero interest; this is converted to 0.5 to preserve the distinction
while still allowing numerical operations. The 100 songs are then ranked by
their peak score and the top 15 are selected, as the majority of remaining songs
score below 5 and would only add visual noise without contributing meaningful
insight.

The chart uses horizontal bars rather than vertical ones because movie titles
are long strings that would be unreadable rotated on a vertical axis. Bars are
sorted from highest to lowest score so the eye immediately reads a hierarchy
from top to bottom. The exact score is printed at the tip of each bar to
eliminate the need to read back against the X axis. A red dashed reference line
at 100 marks the maximum possible Google Trends score, making it instantly clear
which songs reached the absolute peak. Finally, a hover tooltip reveals the full
song name and artist — details that cannot fit directly on the chart — following
the "overview first, details on demand" principle.
```js
const parseScore = s => s === "<1" ? 0.5 : +s;

const processed = raw_dataset.map(d => ({
  song:   d.song_data.song_name,
  artist: d.song_data.artist_names,
  movie:  d.movie_data.title,
  score:  Math.max(...d.trends.map(t => parseScore(t.scores.song)))
}));

const top15 = processed
  .sort((a, b) => b.score - a.score)
  .slice(0, 15);
```
```js
Plot.plot({
  title: "Top 15 songs by Google Trends peak score (France)",
  subtitle: "Score 0–100 : 100 = peak popularity over the full period",
  marginLeft: 220,
  marginRight: 60,
  width: width,
  height: 480,
  x: {
    label: "Google Trends peak score →",
    domain: [0, 105],
    grid: true
  },
  y: {
    domain: top15.map(d => d.movie),
    label: null
  },
  marks: [
    Plot.barX(top15, {
      x: "score",
      y: "movie",
      fill: "#4A90D9",
      sort: { y: "-x" },
      tip: {
        channels: {
          "Film":    "movie",
          "Artiste": "artist",
          "Chanson": d => d.song.length > 50 ? d.song.slice(0, 50) + "…" : d.song,
          "Score":   "score"
        }
      }
    }),
    Plot.text(top15, {
      x: "score",
      y: "movie",
      text: d => d.score.toFixed(0),
      dx: 6,
      textAnchor: "start",
      fontSize: 12,
      fill: "#333"
    }),
    Plot.ruleX([100], {
      stroke: "#e74c3c",
      strokeDasharray: "4,4",
      strokeWidth: 1.5
    })
  ]
})
```