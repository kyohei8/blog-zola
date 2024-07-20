+++
title="Gatsby.jsで年ごと、月ごとで記事一覧を表示したい"
[taxonomies]
tags=["Gatsby.js"]
+++

Gatsby.js でブログなどを書いていると wordpress などにある月別、年別のアーカイブページがないことに気がついたので、自作で作ってみました。

## 要約すると

1. 月別など期間でフィルターするクエリを学ぶ
2. `/2019/` , `/2019/01/` などの月別、年別のアーカイブページを作成する
3. ページ内のクエリで月別、年別で取得するフィルターして記事を取得する

## 年ごと、月ごとでフィルタするクエリ

特定の期間の記事を取得する場合、allMarkdownRemark の`filter`に時間の指定を追加します。例として以下のようなクエリになります。
後にページ呼び出し後のクエリとして利用するので [http://localhost:8000/\_\_\_graphql](http://localhost:8000/___graphql) で動作を確認しておきます。

```graphql
{
  allMarkdownRemark(
    filter: { frontmatter: { date: { gte: "2019-04-01T00:00:00.000Z", lt: "2019-04-30T23:59:59.999Z" } } }
  ) {
    totalCount
    edges {
      node {
        id
        frontmatter {
          date
          title
        }
        fields {
          slug
        }
      }
    }
  }
}
```

## 月別、年別のアーカイブページを作成

まず `/2019/`, `/2019/01/`でアクセスできるページを作成します。

`gatsby-node.js`で以下のようなコードを追加します。

```js
const path = require(`path`);
const _ = require('lodash');

exports.createPages = ({ graphql, actions }) => {
  const { createPage } = actions;

  const periodTemplate = path.resolve('src/templates/period-summary.tsx');

  return graphql(
    `
      {
        allMarkdownRemark(sort: { fields: [frontmatter___date], order: DESC }, limit: 1000) {
          edges {
            node {
              fields {
                slug
              }
              frontmatter {
                title
                tags
                year: date(formatString: "YYYY")
                month: date(formatString: "MM")
              }
            }
          }
        }
      }
    `
  ).then((result) => {
    if (result.errors) {
      throw result.errors;
    }

    // Create blog posts pages.
    const posts = result.data.allMarkdownRemark.edges;

    const years = new Set();
    const yearMonths = new Set();

    // 記事ページを作成
    posts.forEach((post, index) => {
      // 記事のページ等を作成...

      const { year, month } = post.node.frontmatter;

      // 年、年月をまとめる
      years.add(year);
      yearMonths.add(`${year}/${month}`);
    });

    // 年別ページ
    years.forEach((year) => {
      createPage({
        path: `/${year}/`,
        component: periodTemplate,
        context: {
          displayYear: year,
          periodStartDate: `${year}-01-01T00:00:00.000Z`,
          periodEndDate: `${year}-12-31T23:59:59.999Z`
        }
      });
    });

    // 月別ページ
    yearMonths.forEach((yearMonth) => {
      const [year, month] = yearMonth.split('/');
      const startDate = `${year}-${month}-01T00:00:00.000Z`;
      const newStartDate = new Date(startDate);
      // 月末日を取得
      const endDate = new Date(
        new Date(newStartDate.setMonth(newStartDate.getMonth() + 1)).getTime() - 1
      ).toISOString();

      createPage({
        path: `/${year}/${month}/`,
        component: periodTemplate,
        context: {
          displayYear: year,
          displayMonth: month,
          periodStartDate: startDate,
          periodEndDate: endDate
        }
      });
    });

    return null;
  });
};
```

### 解説

まず、すべての記事を取得するクエリに年と月を追加します。
[GraphQL の alias](https://graphql.org/learn/queries/#aliases)を利用して年と月を取り出します。

```graphql
  year: date(formatString: "YYYY")
  month: date(formatString: "MM")
```

取得した記事一覧の年、月を一意にするため、Set 型にまとめます。

```js
const years = new Set();
const yearMonths = new Set();

posts.forEach((post, index) => {
  const { year, month } = post.node.frontmatter;

  years.add(year);
  yearMonths.add(`${year}/${month}`);
});
```

`/2019/`でアクセスできる、年別のページを作成します。
年別のフィルターは 1 月１日から 12 月 31 日までなので、ページのクエリに渡す数値は簡単に定義できます。

```js
// 年別ページ
years.forEach((year) => {
  createPage({
    path: `/${year}/`,
    component: periodTemplate,
    context: {
      displayYear: year,
      periodStartDate: `${year}-01-01T00:00:00.000Z`,
      periodEndDate: `${year}-12-31T23:59:59.999Z`
    }
  });
});
```

`/2019/02/`等でアクセスできる、月別のページを作成します。
`yearMonths`には`2019/01`,`2019/02`,...のような文字列で渡しているので、年と月に分けます。
年と同様に月初から月末までの期間でフィルターを作成します。
月初日は 01 日で固定でいいのですが、月末日は年跨ぎやうるう年があるので少し厄介です。
以下のコードでは次の月初日の 1ms 前の時刻を取得するようにして月末日を取得するようにしています。

```js
yearMonths.forEach((yearMonth) => {
  const [year, month] = yearMonth.split('/');
  const startDate = `${year}-${month}-01T00:00:00.000Z`;
  const newStartDate = new Date(startDate);
  // 月末日を取得
  const endDate = new Date(new Date(newStartDate.setMonth(newStartDate.getMonth() + 1)).getTime() - 1).toISOString();

  createPage({
    path: `/${year}/${month}/`,
    component: periodTemplate,
    context: {
      displayYear: year,
      displayMonth: month,
      periodStartDate: startDate,
      periodEndDate: endDate
    }
  });
});
```

### GraphQL で確認

```graphql
{
  allSitePage {
    totalCount
    edges {
      node {
        path
      }
    }
  }
}
```

のようなクエリでどのようなページが作成されているか確認ができます。(Rails でいうと`rake routes`みたいな)

## ページを作成

`src/templates/`等にページを作成します。
上記でページを生成したとき、pageContext で`periodStartDate`, `periodEndDate`を渡しているので
ページのクエリ上でそれらを利用することができます。
また、表示用に`displayYear`, `displayMonth`も渡しているので、それらは画面に表示します。

```ts
const PeriodSummary: React.FC<PeriodSummaryProps> = ({ pageContext, data, location }) => {
  const { displayMonth, displayYear } = pageContext;
  const { title } = data.site.siteMetadata;
  const { edges, totalCount } = data.allMarkdownRemark;
  return (
    <Layout location={location} title={title}>
      <h2>
        {displayYear}年{displayMonth}月の記事 ({totalCount})
      </h2>
      <ul>
        {edges.map(({ node }) => {
          const { slug } = node.fields;
          const { title, date } = node.frontmatter;
          return (
            <li key={slug}>
              <Link to={slug}>{title}</Link>
              {date}
            </li>
          );
        })}
      </ul>
    </Layout>
  );
};

export const query = graphql`
  query($periodStartDate: Date, $periodEndDate: Date) {
    ...
    allMarkdownRemark(
      limit: 200
      sort: { fields: [frontmatter___date], order: DESC }
      filter: {
        frontmatter: { date: { gte: $periodStartDate, lt: $periodEndDate } }
      }
    ) {
      totalCount
      edges {...}
    }
  }
`;

export default PeriodSummary;
```

これで月別、年別で記事一覧を表示できるようになりました。めでたしめでたし。
