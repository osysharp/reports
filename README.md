# Osysharp.Reports

Reports your team builds for itself, and the overview it opens every morning. Start from one thing your app keeps (an
order, a ticket, a candidate), pick fields by following its relations (Order › Café › Region), group, measure and
filter, and read the answer as figures, a table and a chart. It updates as you pick. Export it as a CSV that says what
the screen says, and save it by name for yourself or for the whole team.

Every report is computed in the database, as the person reading it may read it. However many orders it covers, a report
reads only the aggregate rows it shows, never the orders themselves.

```osy
use Osysharp.Reports@1;
use Osysharp.Explore;
use Osysharp.TimeZone@0;               // the time zone a day and a week are counted in

policy ReadsReports => IsStaff;     // who opens reports
policy SharesReports => IsLead;     // who shares a saved report with the team
policy ManagesClock => IsAdmin;     // who sets the app's time zone (Settings › Time and date)

public class OrderReports : IReportSource {
  public ReportSource Describe() {
    var placed = Explore.Path<Order>(o => o.PlacedAt);
    var shipped = Explore.Path<Order>(o => o.ShippedAt);
    return new ReportSource {
      Key = "orders", Title = "Orders", Root = Explore.Root<Order>(), Noun = "order", Nouns = "orders", Date = placed,
      Measures = [
        new ReportMeasure { Label = "Orders", Measure = new ExploreMeasure { Name = "orders" } },
        new ReportMeasure { Label = "Time to ship", Format = FigureFormat.Duration, HigherIsBetter = false,
                            Measure = new ExploreMeasure { Name = "toShip", Kind = MeasureKind.Median, Field = shipped, Since = placed } },
      ],
      Dimensions = [new ReportDimension { Field = Explore.Path<Order>(o => o.Cafe.Region), Label = "Region" }],
      // What "Show the rows" lists of each order a figure counted.
      RowFields = [Explore.Path<Order>(o => o.Cafe), placed],
    };
  }
}

[Route("/reports")] [Authorize(ReadsReports)]
component OrdersOverview() {
  render { ReportOverview("orders", ["orders", "toShip"], build: "/reports/build"); }
}

[Route("/reports/build")] [Authorize(ReadsReports)]
component OrdersBuilder() {
  render { ReportBuilder("orders"); }
}
```

Your app says what it reports on once, in its own words: a class per root, found without being listed. Every other
field a reader may reach from that root is offered too, from your model.

## What it does

| | |
|---|---|
| **Fields by relation** | one root, and every field reachable from it within three steps, with its trail ("Order › Café › Region"). Search across all of them. A path that crosses a collection says so ("one per line") and never counts a row twice |
| **Measures** | count, total, average, median, any percentile (the 90th is one click), count of different values, earliest and latest; a time between two dates (shipped since placed); a share of one count in another. Your app's own measures come first, in its words |
| **Filters you build by pointing** | pick a field and the filter fits its kind: tick choices, pick records with a search, a number or a duration more than, less than or between, a date range in your app's time zone, words that contain or start with, yes or no, empty or not. Conditions read back as sentences ("Courier is Bike courier or Freight"), and nest in "all of" and "any of" groups |
| **Show groups where** | conditions on the groups themselves, by their measures ("cafés with more than ten orders"), kept apart from the conditions on rows |
| **Periods in your time zone** | today, 7, 30 and 90 days, this week, month, quarter and year, last month, or two days you choose. A day, a week and a month start at midnight in the app's time zone, picked from a list in Settings › Time and date (Osysharp.TimeZone), and follow summer time. A trend has a bar for every day, even a quiet one |
| **The overview** | headline figures with their change against the period before (in the success colour when it is good news, whichever way that is), what stands open now, the trend, and a breakdown by any of your groupings. A figure from a second source joins the breakdown on the grouping of the same name |
| **The rows behind every figure** | press a figure, or a row of a grouped table, and see the rows it counted: "Shipped" for one region lists that region's shipped orders, a day of a trend that day's. Give the source the fields to list (`RowFields`) |
| **CSV** | the same period, the same rows and the same words as the screen, written by one function, so the file and the page cannot disagree |
| **Saved reports** | a report keeps the question, never the numbers. Whoever opens it sees the rows they may read, as they are then. A private one is its owner's; a shared one is everybody's who reads reports |
| **Custom fields as columns** | rows a kit keeps one per record and key, a custom field's answers, are columns of the record: "Ticket › Customer › Their plan" groups, filters and sums as its kind reads. With `Osysharp.CustomFields` it takes one relation in your model |
| **Only what the reader may read** | a field masked for the reader, or a path into something they may read none of, is never offered, and a saved report naming one says so rather than answering a smaller question. A row they may not read counts for nothing |

| The overview | Filters in nested groups |
|---|---|
| ![an overview of orders: figures, a daily trend and a breakdown by region](docs/reports-overview.png) | ![a report with an "all of" group holding an "any of" group](docs/reports-filters.png) |

| Grouped by a relation, in the dark | On a phone |
|---|---|
| ![orders grouped by the café's region, dark theme](docs/reports-builder-dark.png) | ![the overview on a phone, figures two to a line](docs/reports-overview-phone.png) |

## Its tests

The kit's test app is a coffee roaster's wholesale orders: cafés in three regions, a courier, a price list only the
owner may read, and a margin masked from the sales reps. Every figure is checked against an answer worked out by hand:
medians and percentiles, a month in Stockholm across the change to winter time, a week that starts on Sunday, filters
of every kind, nested groups, conditions on groups, the CSV against the screen, saved reports shared and private, and
each reader seeing only their own orders. The builder and the overview are driven as a person would use them, wide and
on a phone, light and dark, with `osy test`, and photographed in a real browser with `osy test --pixels`.

## Examples

Taken from this package's own tests, which compile and run on every release.

```osy
using Osysharp.Explore;
using Osysharp.Reports;
using ReportsKitTests;

[Test(Roastery)]
[runas(Olle)]
void the_csv_is_the_table_row_for_row() {
  var a = RunSourceReport("orders", Orders([Curated("orders"), Curated("value"), Curated("shipMedian")],
                                           groups: [new ExploreGroup { Field = Explore.Path<Order>(o => o.Cafe.Region) }]),
                          PeriodPreset.Last30Days);
  var csv = ReadCsv(ReportCsv(a.Result, OrderSource()));
  Assert.Equal(3, csv.Rows.Count);
  // The headings are the source's words, as the table's header row shows them.
  Assert.Equal("City", csv.Get(csv.Rows[0], "Region"));
  Assert.Equal("3", csv.Get(csv.Rows[0], "Orders"));
  Assert.Equal("4,580", csv.Get(csv.Rows[0], "Value"));
  Assert.Equal("24 h", csv.Get(csv.Rows[0], "Time to ship"));
  // …and every cell is the table's cell.
  for (var i = 0; i < a.Result.Rows.Count; i = i + 1) {
    var cells = RowText(a.Result, i).Split(" | ");
    for (var j = 0; j < cells.Length; j = j + 1) {
      Assert.Equal(cells[j], csv.Get(csv.Rows[i], HeadingOf(a.Result.Columns[j], OrderSource())));
    }
  }
}
```

From [`tests/csv.test.osy`](tests/csv.test.osy).

```osy
using Osysharp.Explore;
using Osysharp.Reports;
using ReportsKitTests;

[Test(Roastery)]
[runas(Olle)]
void a_groups_rows_are_the_rows_its_figure_counted() {
  var q = Orders([Curated("shipped"), Curated("orders")],
                 groups: [new ExploreGroup { Field = Explore.Path<Order>(o => o.Cafe.Region) }]);
  var a = RunSourceReport("orders", q, PeriodPreset.Last30Days);
  var coast = a.Result.Rows.FindIndex(r => r.Cells[0].Text == "Coast");
  Assert.Equal("Coast | 1 | 2", RowText(a.Result, coast));
  var shipped = RunSourceRows("orders", q, "shipped", PeriodPreset.Last30Days, columns: a.Result.Columns, row: a.Result.Rows[coast]);
  Assert.Equal("", shipped.Result.Problem);
  Assert.Equal(1, shipped.Result.Rows.Count);
  Assert.Equal("Fjord Coffee", shipped.Result.Rows[0].Cells[0].Text);
  var placed = RunSourceRows("orders", q, "orders", PeriodPreset.Last30Days, columns: a.Result.Columns, row: a.Result.Rows[coast]);
  Assert.Equal(2, placed.Result.Rows.Count);
}
```

From [`tests/drill.test.osy`](tests/drill.test.osy).

```osy
using Osysharp.Explore;
using Osysharp.Reports;
using ReportsKitTests;

[Test(Roastery)]
[runas(Olle)]
void the_curated_totals_for_the_last_30_days_are_the_septembers_seven_orders() {
  var a = RunSourceReport("orders", Orders([Curated("orders"), Curated("kilos"), Curated("value"), Curated("shipped"),
                                            Curated("shippedShare"), Curated("shipMedian"), Curated("shipP90")]),
                          PeriodPreset.Last30Days);
  Assert.Equal("", a.Result.Problem);
  Assert.Equal("1 Sep – 30 Sep 2026", a.Period);
  // 7 orders · 62 kg · 14,680 kr · 4 shipped · 4 of 7 shipped · shipped after 6, 24, 24 and 48 hours: the median is 24 h,
  // and the 90th percentile 24 + 0.7 × 24 = 40.8 hours.
  Assert.Equal("7 | 62 | 14,680 | 4 | 57.1% | 24 h | 40 h 48 min", RowText(a.Result, 0));
}
```

From [`tests/figures.test.osy`](tests/figures.test.osy).

## Learn more

- [The Osysharp.Reports reference](https://osysharp.com/reference/ui/reports-kit/)
- [Osysharp.Reports in the package catalogue](https://osysharp.com/packages/#osysharp-reports)
- [The Osy# reference](https://osysharp.com/reference/) — the language, the runtime and every kit
- [`llms-full.txt`](https://osysharp.com/llms-full.txt) — the whole reference and specification as one Markdown file, for a coding agent

## Licence

MIT — see [`LICENSE`](LICENSE).
