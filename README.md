# broccoli-test-helper

Some test helpers for testing BroccoliPlugins build and rebuild behavior dead simple and expect diff friendly.

```js
import { expect } from "chai";
import { buildOutput, createTempDir } from "broccoli-test-helper";
import MyBroccoliPlugin from "../index";

describe("MyBroccoliPlugin", () => {
  let input;

  beforeEach(() => createTempDir().then(tempDir => {
    input = tempDir;
  }));

  afterEach(() => input.dispose());

  it("should build", () => {
    input.write({
      "index.js": `export { A } from "./lib/a";`
      "lib": {
        "a.js": `export class A {};`,
        "b.js": `export class B {};`,
        "c.js": `export class C {};`
      }
    });

    return buildOutput(
      new MyBroccoliPlugin(input.path())
    ).then(output => {
      expect(
        output.read()
      ).to.deep.equal({
        "index.js": `exports.A = require("./lib/a").A;`,
        "lib": {
          "a.js": `exports.A = class A {};`,
          "b.js": `exports.B = class B {};`,
          "c.js": `exports.C = class C {};`
        }
      });

      // rebuild noop
      return output.rebuild();
    }).then(output => {
      expect( output.changes() ).to.deep.equal({ });

      input.write({
        "index.js": "export class A {};",
        "lib": null // delete dir
      });

      // rebuild changes
      return output.rebuild();
    }).then(output => {
      expect( output.changes() ).to.deep.equal({
        "lib/c.js": "unlink",
        "lib/b.js": "unlink",
        "lib/a.js": "unlink",
        "lib/":     "rmdir",
        "index.js": "change"
      });

      expect( output.read() ).to.deep.equal({
        "index.js": `exports.A = class A {};`
      });
    });
  });
});
```
