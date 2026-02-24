---
title: Movie Song Trends
---

# Movie & Soundtrack Trends Dashboard

This dashboard was created as a group project exploring the relationship between
movies and their soundtracks through Google Trends data. The dataset contains
100 movies and series shown in France, each paired with a notable song from its
soundtrack, with monthly search interest scores ranging from 0 to 100.

---

```js
const raw_dataset = await FileAttachment("data/movie-song-trends_dataset.json").json();
```

## Graph 1 — How does the popularity lifecycle of a cinematic release compare to that of its official soundtrack over time?

**Objective:** To visually identify the "Crossover Point"—the exact moment a derivative work (the soundtrack) surpasses the original work (the movie) in public interest—and to analyze the decay rate of both cultural artifacts.

### How this chart was built

### Data Provenance & Structure

The analysis relies on a compiled JSON dataset structured across three primary dimensions:

* **Movie Metadata:** Title, release date, and global popularity metrics.
* **Audio Metadata:** Track title, artist, and release date.
* **Time-Series Data (Trends):** A monthly chronological array tracking search interest scores. 

*Note on Data Integrity:* Background statistical noise (originally logged as less than 1) has been strictly converted to zeroes to allow for algorithmic processing. Furthermore, an upstream filter has been deployed to exclude movie-song pairs where the musical signal remains at absolute zero throughout the entire timeline, ensuring only relevant samples are analyzed.

### Mathematical Processing: Min-Max Normalization

Comparing raw search volumes between a global blockbuster and a single audio track introduces a mathematical bias, as the movie's massive volume visually flattens the music's data. To counteract this, a relative normalization was applied to each time series.

Each data point was scaled to a 0 to 100 percent range relative to its own absolute maximum peak. This transformation allows us to study the pure dynamics (the cycles of hype and cultural decay) rather than incomparable absolute volumes.

### Global Average Mode

To contextualize the data, users can select **"Global Average"** from the target selector. This aggregates all valid movies and soundtracks, calculates their mean curves, and dynamically normalizes them to a 100 percent scale to represent the definitive industry standard comparison between film and music longevity.

### Visual Encoding & Interactive Features

* **Geometric Marks:** Continuous lines (Plot lineY) were selected for time-series data. The difference mark (Plot differenceY) fills the area between the two curves: green for movie dominance, orange for soundtrack dominance.
* **Target Selector:** A dynamic dropdown menu allows the user to isolate specific movies or analyze the overall industry average.
* **Hover Telemetry:** The implementation of an interactive pointer generates a vertical tracking line equipped with a tooltip, providing an immediate readout of the exact relative percentages.

```js
const selectedMovieTitle = view(Inputs.select(
  ["Global Average"].concat(
    raw_dataset.filter(d => {
      return d.trends.some(t => {
        const score = t.scores.song;
        return score !== "<1" && Number(score) > 0;
      });
    }).map(d => d.movie_data.title)
  ), 
  {
    label: "Select a target:",
    value: "Global Average"
  }
));
```

```js
const formattedData = (() => {
  const parseScore = (scoreStr) => scoreStr === "<1" ? 0 : Number(scoreStr);
  const validMovies = raw_dataset.filter(d => d.trends.some(t => parseScore(t.scores.song) > 0));

  if (selectedMovieTitle === "Global Average") {
    const allNorm = validMovies.map(movie => {
      const maxM = Math.max(...movie.trends.map(t => parseScore(t.scores.movie)));
      const maxS = Math.max(...movie.trends.map(t => parseScore(t.scores.song)));
      return movie.trends.map(t => ({
        date: t.date,
        moviePct: maxM > 0 ? (parseScore(t.scores.movie) / maxM) * 100 : 0,
        songPct: maxS > 0 ? (parseScore(t.scores.song) / maxS) * 100 : 0
      }));
    });

    const dates = validMovies[0].trends.map(t => t.date);
    const averaged = dates.map((dateStr, i) => {
      return {
        date: new Date(dateStr),
        avgMovie: d3.mean(allNorm, d => d[i].moviePct),
        avgSong: d3.mean(allNorm, d => d[i].songPct)
      };
    });

    const maxAvgM = Math.max(...averaged.map(d => d.avgMovie));
    const maxAvgS = Math.max(...averaged.map(d => d.avgSong));

    return averaged.map(d => ({
      date: d.date,
      movieRelativePop: maxAvgM > 0 ? (d.avgMovie / maxAvgM) * 100 : 0,
      songRelativePop: maxAvgS > 0 ? (d.avgSong / maxAvgS) * 100 : 0,
      rawMovieScore: "N/A (Average Mode)",
      rawSongScore: "N/A (Average Mode)"
    }));

  } else {
    const targetData = raw_dataset.find(d => d.movie_data.title === selectedMovieTitle);
    
    let parsed = targetData.trends.map(d => ({
      date: new Date(d.date),
      rawMovieScore: parseScore(d.scores.movie),
      rawSongScore: parseScore(d.scores.song)
    }));

    const maxMovie = Math.max(...parsed.map(d => d.rawMovieScore));
    const maxSong = Math.max(...parsed.map(d => d.rawSongScore));

    return parsed.map(d => ({
      ...d,
      movieRelativePop: maxMovie > 0 ? (d.rawMovieScore / maxMovie) * 100 : 0,
      songRelativePop: maxSong > 0 ? (d.rawSongScore / maxSong) * 100 : 0
    }));
  }
})();
```

```js
Plot.plot({
  title: `Normalized Comparative Analysis: ${selectedMovieTitle}`,
  subtitle: selectedMovieTitle === "Global Average" ? "Industry Average: Movies vs. Soundtracks Peak Intensity" : "Movie vs. Soundtrack Peak Intensity",
  width: width,
  height: 500,
  marginLeft: 50,
  // Injection du module de légende
  color: {
    legend: true,
    type: "categorical",
    domain: [
      "Movie Popularity", 
      "Soundtrack Popularity", 
      "Movie Dominates", 
      "Soundtrack Dominates"
    ],
    range: [
      "#2ca02c",                 // Vert solide (Ligne Film)
      "#ff7f0e",                 // Orange solide (Ligne Musique)
      "rgba(44, 160, 44, 0.4)",  // Vert transparent (Zone Film)
      "rgba(255, 127, 14, 0.4)"  // Orange transparent (Zone Musique)
    ]
  },
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
      positiveFill: "rgba(255, 127, 14, 0.4)", // Positif (Musique > Film) -> Orange
      negativeFill: "rgba(44, 160, 44, 0.4)"   // Négatif (Film > Musique) -> Vert
    }),
    
    Plot.lineY(formattedData, { x: "date", y: "movieRelativePop", stroke: "#2ca02c", strokeWidth: 2 }),
    Plot.lineY(formattedData, { x: "date", y: "songRelativePop", stroke: "#ff7f0e", strokeWidth: 2 }),
    
    Plot.ruleX(formattedData, Plot.pointerX({
      x: "date", stroke: "gray", strokeWidth: 1, strokeDasharray: "4,4",
      channels: {
        "Movie Rel. Pop. (%)": d => Math.round(d.movieRelativePop),
        "Song Rel. Pop. (%)": d => Math.round(d.songRelativePop)
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
