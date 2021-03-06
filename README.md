# observable-prerender

Pre-render Observable notebooks with Puppeteer! Inspired by [d3-pre](https://github.com/fivethirtyeight/d3-pre)

## Why tho

Observable notebooks run in the browser and use browser APIs, like SVG, canvas, webgl, and so much more. Sometimes, you may want to script or automate Observable notebooks in some way. For example, you may want to:

- Create a bar chart with custom data
- Generate a SVG map for every county in California
- Render frames for a MP4 screencast of a custom animation

If you wanted to do this before, you'd have to manually open a browser, re-write code, upload different file attachments, download cells, and repeat it all many times. Now you can script it all!

## Examples

Check out `examples/` for workable code.

### Create a bar chart with your own data

```javascript
const { load } = require("@alex.garcia/observable-prerender");
const notebook = await load("@d3/bar-chart", ["chart", "data"]);
const data = [
  { name: "alex", value: 20 },
  { name: "brian", value: 30 },
  { name: "craig", value: 10 },
];
await notebook.redefine("data", data);
await notebook.screenshot("chart", "bar-chart.png");
await notebook.browser.close();
```

Result:
![Screenshot of a bar chart with 3 bars, with labels "alex", "brian" and "craig", with values 20, 30, and 10, respectively.](https://user-images.githubusercontent.com/15178711/86563267-ee847580-bf18-11ea-9b58-8c5ee6d710f4.png)

### Create a map of every county in California

```javascript
const { load } = require("@alex.garcia/observable-prerender");
const notebook = await load(
  "@datadesk/base-maps-for-all-58-california-counties",
  ["chart"]
);
const counties = await notebook.value("counties");
for await (let county of counties) {
  await notebook.redefine("county", county.fips);
  await notebook.screenshot("chart", `${county.name}.png`);
  await notebook.svg("chart", `${county.name}.svg`);
}
await notebook.browser.close();
```

Some of the resulting PNGs:

| County      | Screenshot                                                                                                                                             |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Los Angeles | ![](https://user-images.githubusercontent.com/15178711/86563356-15db4280-bf19-11ea-86e2-664c64a1593a.png)                                              |
| Merced      | ![Picture of a simple map of Merced county.](https://user-images.githubusercontent.com/15178711/86563375-1e337d80-bf19-11ea-9bc9-03517bb82bab.png)     |
| Sacramento  | ![Picture of a simple map of Sacramento county.](https://user-images.githubusercontent.com/15178711/86563392-25f32200-bf19-11ea-9c96-54e394012585.png) |
| San Diego   | ![Picture of a simple map of Merced county.](https://user-images.githubusercontent.com/15178711/86563413-2ee3f380-bf19-11ea-87c3-5fd08ad0861d.png)     |

### Create frames for an animated GIF

Create PNG frames with `observable-prerender`:

```javascript
const { load } = require("@alex.garcia/observable-prerender");
const notebook = await load("@asg017/sunrise-and-sunset-worldwide", [
  "graphic",
  "controller",
]);
const times = await notebook.value("times");
for (let i = 0; i < times.length; i++) {
  await notebook.redefine("timeI", i);
  await notebook.waitFor("controller");
  await notebook.screenshot("graphic", `sun${i}.png`);
}
await notebook.browser.close();
```

Then use something like ffmpeg to create a MP4 video with those frames!

```bash
 ffmpeg.exe -framerate 30 -i sun%03d.png -c:v libx264  -pix_fmt yuv420p out.mp4
```

Result (as a GIF, since GitHub only supports gifs):

![Screencast of a animation of sunlight time in Los Angeles during the year.](https://user-images.githubusercontent.com/15178711/86563817-ed077d00-bf19-11ea-9922-52ef0fd5c38d.gif)

## Install

```bash
npm install @alex.garcia/observable-prerender
```

## API Reference

Although not required, a solid understanding of the Observable notebook runtime and the embedding process could help greatly when building with this tool. Here's some resources you could use to learn more:

- [How Observable Runs from Observable](https://observablehq.com/@observablehq/how-observable-runs)
- [Downloading and Embedding Notebooks from Observable](https://observablehq.com/@observablehq/downloading-and-embedding-notebooks)

### prerender.**load**(notebook, _targets_)

Load the given notebook into a page in a browser. `notebook` is the id of the notebook on observablehq.com, like `@d3/bar-chart` or `@asg017/bitmoji`. For unlisted notebooks, be sure to include the `d/` prefix (e.g. `d/27a0b05d777304bd`). `targets` is an array of cell names that will be evaluated. Every cell in `targets` (and the cells they depend on) will be evaluated and render to the page's DOM. If not supplied, then all cells (including anonymous ones) will be evaluated by default.

This returns a Notebook object. A Notebook has `page` and `browser` properties, which are the Puppeteer page and browser objects that the notebook is loaded with. This gives a lower-level API to the underlying Puppeteer objects that render the notebook, in case you want more fine-grain API access for more control.

### notebook.**value**(cell)

Returns a Promise that resolves value of the given cell for the book. For example, if the `@d3/bar-chart` notebook is loaded, then `.value("color")` would return `"steelblue"`, `.value("height")` would return `500`, and `.value("data)` would return the 26-length JS array containing the data.

Keep in mind that the value return is serialized from the browser to Node, see below for details.

### notebook.**redefine**(cell, value)

Redefine a specific cell in the Notebook runtime to a new value. `cell` is the name of the cell that will be redefined, and `value` is the value that cell will be redefined as. If `cell` is an object, then all of the object's keys/values will be redefined on the notebook (e.g. `cell={a:1, b:2}` would redefine cell `a` to `1` and `b` to `2`).

Keep in mind that the value return is serialized from the browser to Node, see below for details.

### notebook.**screenshot**(cell, path, _options_)

Take a screenshot of the container of the element that contains the rendered value of `cell`. `path` is the path of the saved screenshot (PNG), and `options` is any extra options that get added to the underlying Puppeteer `.screenshot()` function ([list of options here](https://pptr.dev/#?product=Puppeteer&version=v5.0.0&show=api-pagescreenshotoptions)). For example, if the `@d3/bar-chart` notebook is loaded, `notebook.screenshot('chart)

### notebook.**svg**(cell, path)

If `cell` is a SVG cell, this will save that cell's SVG into `path`, like `.screenshot()`. Keep in mind, the browser's CSS won't be exported into the SVG, so beware of styling with `class`.

### notebook.**waitFor**(cell, _status_)

Returns a Promise that resolves when the cell named `cell` is `"fulfilled"` (see the Observable inspector documentation for more details). The default is fulfilled, but `status` could also be `"pending"` or `"rejected"`. Use this function to ensure that youre redefined changes propagate to dependent cells.

## A Note on Serialization

There is a Puppeteer serialization process when switching from browser JS data to Node. Returning primitives like arrays, plain JS objects, numbers, and strings will work fine, but custom objects, HTML elements, Date objects, and some typed arrays may not. Which means that some methods like `.value()` or `.redefine()` may be limited or may not work as expected, causing subtle bugs. Check out the [Puppeteer docs](https://pptr.dev/#?product=Puppeteer&version=v3.1.0&show=api-pageevaluatepagefunction-args) for more info about this.
