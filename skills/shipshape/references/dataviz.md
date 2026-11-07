# Data visualisation

## Form before colour

| You have | Use |
|---|---|
| One number | A stat tile: the number large, a label, and a comparison (vs last period, vs target) |
| A few headline numbers | A KPI row |
| Magnitude across categories | Bars (horizontal when labels are long), sorted unless the order means something |
| Change over time | A line (bars for few, discrete periods) |
| Part of a whole | Stacked bar; a pie or donut only for at most five clearly different slices |
| Positive vs negative | Diverging bars around zero |
| Relationship | Scatter |
| Before and after | Dumbbell or slope chart |
| More than about seven classes that all matter | A table |
| One series is the story | Emphasis: that series in the accent, the rest in grey |

Dashboards read top to bottom: KPI strip, then the main chart, then supporting charts, then the detail table.

## Honesty

- Bars start at zero.
- One y-axis only. Never dual axes: use two charts or index both series to 100.
- No 3D, no perspective, no decorative depth.
- Compared panels and small multiples share identical scales.
- No invented precision; round to what the data supports.

## Colour by job

- **Categorical:** a fixed hue order, assigned to entities, not ranks, so filtering never repaints the remaining series. Never cycle hues; fold extra categories into "Other" or facet the chart.
- **Sequential:** one hue, light to dark.
- **Diverging:** two opposing hues with a neutral grey middle.
- **Status:** reserved colours that always come with an icon and a label.
- No rainbows.
- Validate numerically rather than by eye: adjacent series should stay distinguishable under colour-blindness simulation (a perceptual difference of about 8+ in OKLab, about 15+ for normal vision), and marks need 3:1 contrast with the background. Scatter plots and maps, where every pair can touch, cap out around three series. Dark mode gets its own validated steps.

## Chrome and labels

- Recessive chrome: solid hairline gridlines (or none), no chart borders or backgrounds, thin marks (lines about 2px, bars no thicker than they need to be), small gaps between touching fills instead of strokes.
- The title states the finding ("Claims fell 18% after launch"), not the metric name.
- Label selectively: endpoints, extremes, the point of the story.
- A legend whenever there are two or more series; add direct labels as well when there are four or fewer.
- Units on axes, a muted source line, consistent number formats.
- Text never takes the series colour; use ink with a swatch beside it.
- Tabular figures in columns and axis ticks; proportional figures for hero numbers. Nothing clipped.
- Data UI uses the product's sans; keep expressive display faces for marketing and editorial.

## Interaction

- Tooltip on hover and focus (crosshair for time series), hit areas of 24px or more, keyboard parity.
- A table view of the same data, so nothing is only available inside a tooltip.
- Filters in one row above the charts they affect.
- On refresh, keep the previous data dimmed instead of blanking the chart.
