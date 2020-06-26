---
title: "RxJS: Reporting API Usage"
description: How to help the core team by reporting your project's API usage
date: "2020-06-22T15:22:00+1000"
categories: []
keywords: []
ckTags: ["1464979"]
cardImage: "./title-card.jpeg"
---

![Tape measure](title.jpeg "Photo by Siora Photography on Unsplash")

RxJS API usage is a topic that's often discussed at the core-team meetings. The discussion usually involves speculation regarding the most-used and least-used parts of the API. The API is rather large — there are many operators and a considerable number of observable creators, too.

The topic was raised again, last week, with team members lamenting that there wasn't a simple way of collecting anonymous usage statistics. Locating a project's RxJS API use is something that's done in the [RxJS Tools](https://rxjs.tools) that I'm working on — the tools include TypeScript and Babel transforms that instrument development builds — so I set aside some time to write a small package that collects and reports anonymous API usage statistics via a Google Form.

The package is [`rxjs-report-usage`](https://github.com/cartant/rxjs-report-usage).

When run inside a project, the package's script locates all JavaScript and TypeScript files - except for those in the `node_modules` directory - and parses them with Babel — which, BTW, has [excellent documentation](https://github.com/jamiebuilds/babel-handbook/blob/master/translations/en/plugin-handbook.md). The parsed code is searched for `import` statements and `require` calls that consume `rxjs` and a usage count is recorded for each consumed RxJS API.

The script also locates any `rxjs` and `typescript` packages within `node_modules` and reports their versions. It only does this for `rxjs` and `typescript`; the versions of other packages are not included in the report.

The payload of usage statistics looks like this and is presented to the developer before being sent to the core team (no information is sent without the developer's consent):

```json
{
  "apis": {
    "rxjs": {
      "concat": 1,
      "merge": 1,
      "of": 4
    },
    "rxjs/operators": {
      "concatMap": 1,
      "mergeMap": 1
    }
  },
  "packageVersions": {
    "rxjs": ["6.5.5"],
    "typescript": ["3.9.5"]
  },
  "schemaVersion": 1,
  "timestamp": 1592659729551
}
```

After the developer's confirmation, the reported usage statistics are posted to a Google Form. And, once we have a reasonable data set, we'll most likely use something like the [`ndjson-cli`](https://github.com/mbostock/ndjson-cli) to filter and query the JSON records.

The information should give us some insight into which parts of the API are used most and which are used least.

If you'd like to help us collect this usage information, you can do so by running the package's script in each of your RxJS-consuming projects. There are several ways that you can do this:

- If you are using the most-recent versions of either the [ESLint](https://github.com/cartant/eslint-plugin-rxjs) or [TSLint](https://github.com/cartant/rxjs-tslint-rules) RxJS rules, `rxjs-report-usage` will already be installed — as a dependency — and you can run it using npm:

        npm rxjs-report-usage

  or yarn:

        yarn rxjs-report-usage

  You'll also have the package installed if you are using the most-recent versions of any of the following packages: [`rxjs-etc`](https://github.com/cartant/rxjs-etc), [`rxjs-marbles`](https://github.com/cartant/rxjs-marbles) or [`rxjs-spy`](https://github.com/cartant/rxjs-spy).

- If you're not using a package that includes `rxjs-report-usage` as a dependency, you can install `rxjs-report-usage` into your project as a `devDependency` and can then run the script as shown above.
- Alternatively, you can run the script directly using `npx`, like this:

        npx rxjs-report-usage

Hopefully, the package will be seen as a safe and convenient way of reporting anonymous API usage statistics to the core team. If you're able to help by reporting your API usage, that'd be awesome. And if you encounter any problems with the package, please [open an issue](https://github.com/cartant/rxjs-report-usage/issues) in the repo.
