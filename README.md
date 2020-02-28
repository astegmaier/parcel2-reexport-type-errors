# parcel2-reexport-type-errors

This repo demonstrates [an issue](https://github.com/parcel-bundler/parcel/issues/4240) where `parcel` throws unhelpful error messages in cases where you re-export a type in a typescript project.

## Repro steps

1. Install project dependencies by running `yarn`
2. Run parcel by running `yarn build`

You'll see this error:

```
× Build failed.
@parcel/packager-js: src\Person.ts does not export 'Person'
C:\<myProject>\src\index.ts
```

## Analysis

The error is caused by the fact that we have one file that exports a type...

***Person.ts***
```ts
export interface Person { firstName: string; lastName: string }
```

...and another file that imports it, and the _re_-exports it...

***index.ts***
```ts
import { Person } from './Person';
...
export { Person }
```

Parcel will fail to remove the `Person` import from `index.ts` in the output JavaScript. When do scope-hoisting, fact that we're importing something that doesn't exist will throw an error. This is by-design behavior - because our transformers run one-file-at-a-time (`tsc` uses the `isolatedModules` flag, and Babel does this by default), it is impossible, when compiling `index.ts`, to tell whether `Person` is a type (which should be stripped away) or a value (which should remain).

The right way for the user to fix this situation is to use [a new feature in Typescript 3.8 that allow you to explicitly say that you want to import a thing as a type, not a value](https://devblogs.microsoft.com/typescript/announcing-typescript-3-8/#type-only-imports-exports).

`tsc` is able to throw a much more helpful error (you can see this by running `yarn typecheck`):

```
× Build failed.
@parcel/validator-typescript: Re-exporting a type when the '--isolatedModules' flag is provided requires using 'export type'.
C:/<myProject>/index.ts
C:/<myProject>/index.ts:7:10
  6 | 
> 7 | export { Person }
>   |          ^^^^^^^ Re-exporting a type when the '--isolatedModules' flag is provided requires using 'export type'.
```

It would be great if parcel also threw this more-helpful error by default.
