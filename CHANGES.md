# node-zutil changelog

## not yet released

(nothing yet)

## 2.0.1

- Fix package.json to publish binding.gyp necessary to have the C bindings
  actually build on `npm install`.

## 2.0.0

- A re-write a this module to work with node v6 and later. This module is almost
  completely different. Changes:

    - Works with node v6 and drops support for earlier versions. Earlier
      versions only worked with node versions up to v0.10.
    - Drops some APIs:
        - `listZones`: Not used in Triton/Manta stack. Not a documented part
          of zone.h.
        - `getZoneAttribute`, `getZoneAttributes`: Not documented stable API
          (none of libzonecfg.h is). I ran into crashes implementing this
          with N-API and node v6. I suspect libzonecfg and its usage of
          libxml2 is just not threadsafe. Also this isn't heavily used by
          Triton/Manta. See [the
          PR](https://github.com/joyent/node-zutil/pull/8) and [the
          branch](https://github.com/joyent/node-zutil/tree/getzoneattr) for
          the crashing attempt.  See workaround in the migration table below.
    - Renames all the other APIs. The new names match the name of the C
      functions, where those exist.

### Migrating from zutil 0.x and 1.x to 2.x

| old API                                                      | new API |
| -------------------------------------------------------------| ------- |
| `getZone() -> {id: <id>, name: <zonename>}`                  | `getzoneid() -> <id>`, `getzonename() -> <zonename>` |
| `getZoneByName(zonename) -> {id: <id>, name: <zonename>}`    | `getzoneidbyname(zonename) -> <id>` |
| `getZoneById() -> {id: <id>, name: <zonename>}`              | `getzonenamebyid(id) -> <zonename>` |
| `getZoneState(zonename) -> '<state>'`                        | `getzonestate(<zonename>) -> '<state>'` |
| `listZones()`                                                | dropped, exec `zoneadm list -p` and parse |
| `getZoneAttribute(zonename, attrname, function (err, attr))` | dropped, exec `zonecfg -z <zonename> attr name=<attrname>`, see `getZoneAttr` below |
| `getZoneAttributes(zonename, function (err, attrs))`         | dropped, exec `zonecfg -z <zonename> attr` and parse |

### Replacement `getZoneAttribute` JS code

```javascript
var assert = require('assert-plus');
var forkExecWait = require('forkexec').forkExecWait;
var VError = require('verror');

/*
 * Get a zonecfg attribute.
 *
 * This calls back with `function (err, attr)` with one of the following:
 * - `function(<error>, undefined)` if there was some error. For example it
 *   is an error if the given `zonename` is not a configured zone.
 * - `function(null, null)` if the zonename exists, but it has no such attr.
 * - `function(null, <attr>)` where attr is an object of the form:
 *
 *          {
 *              name: "<the given attrname>",
 *              type: "<an attribute type, e.g. 'string'>",
 *              value: "<the attribute value>"
 *          }
 */
function getZoneAttr(zonename, attrname, cb) {
    // Attr output from `zonecfg -z ZONE info attr name=NAME` looks like:
    //     "attr:\n\tname: <name>\n\ttype: <type>\n\tvalue: <value>\n"
    var attrRe = /^attr:\n\tname: (.*?)\n\ttype: (.*?)\n\tvalue: (.*?)\n$/m;

    var argv = ['/usr/sbin/zonecfg', '-z', zonename, 'info',
        'attr', 'name=' + attrname];
    forkExecWait({argv: argv, includeStderr: true}, function (err, info) {
        if (err) {
            cb(err);
        } else if (info.stdout === 'No such attr resource.\n') {
            cb(null, null);
        } else {
            var parsed = attrRe.exec(info.stdout);
            if (!parsed) {
                cb(new VError('unexpected zonecfg attr output: %j',
                    info.stdout));
            } else {
                assert.equal(parsed[1], attrname);
                attr = {name: parsed[1], type: parsed[2], value: parsed[3]};
                cb(null, attr);
            }
        }
    });
}

var zonename = '5d4f7599-a991-6b35-dd44-d91936957a6b';
var attrname = 'owner-uuid';
getZoneAttr(zonename, attrname, function (err, attr) {
    console.log('err:', err)
    console.log('attr:', attr)
});
```


## 1.0.0

- joyent/node-zutil#4 Added support for building with node 0.10.

## 0.2.1

- Exclude the "build/" dir from the published npm package.

## 0.2.0

(Use the commit history.)
