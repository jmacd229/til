# Server Side Rendered Sites with 3rd Party Libraries Attempting to Access Window
Because server side rendered sites are built in a Node runtime and generate static HTML to be deployed to a server, there is no access to browser specific properties such as `Window`. This is the case for many SSR frameworks such as Gatsby. Usually calls to `Window` can be controlled within your own application, but importing libraries that call to `Window` can be difficult to workaround.

For my examples below, I am going to discuss my difficulty/learning opportunity I had when working with [Leaflet.js](https://leafletjs.com/) in Gatsby.

## Discovering the issue
Unfortunately, sometimes (especially in my case above) the console won't always tell you explicitly that the cause of the issue was a call to an undefined window. For that reason, I would recommend *running a build after every package added*. I'm usually working in Develop mode in Gatsby which actually has access to `Window` so will not flag the issue immediately, in my case, when I finally ran `gatsby build` this is what I saw as the reason for the failing build

```
failed Building static HTML for pages

 ERROR #95313 

Building static HTML failed
  
  - stylis.esm.js:345 
    node_modules/@emotion/stylis/dist/stylis.esm.js:345:1
```

Which led me to believe their was an issue with the `@emotion` library (this was simply a dependency and not something I was even using directly) and this [Stack Overflow question about a similar issue](https://stackoverflow.com/questions/64646291/gatsby-build-webpack-fail-with-stylis) that was ultimately unhelpful and drove me further down the rabbit hole. I had to go backwards through each of my commits until I finally found the offending commit that caused the build to start failing. This would of course also be fixed with a better CI/CD pipeline, but sometimes when working on small personal projects we don't have access to that.

*TLDR: When working in SSR land, always run a build after installing new libraries if you don't have a CI/CD pipeline to rely on.*


## Discovering the solution

### Using a null loader
As mentioned above, I was able to weed out the offending library, however, I still needed to use this library so I couldn't just remove it altogether. [One option](https://www.gatsbyjs.com/docs/debugging-html-builds/#fixing-third-party-modules) is to replace the module with a "dummy" module in webpack during build time. This allows the build process to skip adding the module to the bundle, and it will be lazy loaded as needed.

#### When a null loader doesn't work
When I used this for my issue above, I tried null loading both `leaflet` and `react-leaflet` (I was using a function from the base `leaflet` in order to customize the pin marker icon on the map).
```javascript
exports.onCreateWebpackConfig = ({ stage, loaders, actions }) => {
  if (stage === "build-html" || stage === "develop-html") {
    actions.setWebpackConfig({
      module: {
        rules: [
          {
            test: /leaflet|react-leaflet/,
            use: loaders.null(),
          },
        ],
      },
    })
  }
}
```
This worked fine for `react-leaflet` since I was just using the jsx it provided, the build was able to run, however I was using a function from `leaflet` so using a null loader there was causing me to see
```
WebpackError: TypeError: (0 , leaflet__WEBPACK_IMPORTED_MODULE_1__.icon) is not a function
```
because the library wasn't loaded, it didn't recognize the function.

The solution to this is to then also use `typeof window !== "undefined"`
This check allows the compiler to know if the window is not currently available, don't traverse the next but of code, you can wrap your entire component with this by doing
```typescript
const ContactMap = (props): ReactElement =>
  typeof window !== 'undefined' && (
```

#### When the null loader and typeof window !== 'undefined' both don't work
In my case I was __still__ seeing the error, this was most likely because I stopped null loading `leaflet`, which I should not have done, and instead used it in conjunction with the window check.
However, this did allow me to find a unique solution that could be useful for other use cases. What I did instead was still loaded `leaflet`, but the problem was that importing it directly via
```javascript
import L from 'leaflet';
```
would immediately throw an error, even if I didn't use `L`. This was most likely due to them referencing `window` since this library was not created with SSR in mind.
That meant I had to find a way to import the library only after the window was available, which was how I came up with the following solution
```typescript
const L =
  typeof window !== 'undefined'
    ? require('leaflet')
    : {
        icon: () => {
          return null;
        },
      };
```
This basically said that if the window wasn't defined then create a fake function for what I need that simply does nothing, but if the window __is__ available, then we can import the library and use the real function.