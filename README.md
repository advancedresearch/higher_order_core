# Higher Order Core

This crate contains core structs and traits for programming with higher order data structures.

### Introduction to higher order data structures

A higher order data structure is a generalization of an ordinary data structure.

In ordinary data structures, the default way of programming is:

- Use data structures for data
- Use methods/functions for operations on data

In a higher order data structure, data and functions become the same thing.

The central idea of a higher order data structure,
is that properties can be functions of the same type.

For example, a `Point` has an `x`, `y` and `z` property.
In ordinary programming, `x`, `y` and `z` might have the type `f64`.

If `x`, `y` and `z` are functions from `T -> f64`,
then the point type is `Point<T>`.

A higher order `Point<T>` can be called, just like a function.
When called as a function, `Point<T>` returns `Point`.

However, unlike functions, you can still access properties of `Point<T>`.
You can also define methods and overload operators for `Point<T>`.
This means that in a higher order data structure, data and functions become the same thing.

### Motivation of programming with higher order data structures

The major application of higher order data structures is geometry.

A typical usage is e.g. to create procedurally generated content for games.

Higher order data structures is about finding the right balance between
hiding implementation details and exposing them for various generic algorithms.

For example, a circle can be thought of as having the type `Point<f64>`.
The argument can be an angle in radians, or a value in the unit interval `[0, 1]`.

Another example, a line can be thought of as having the type `Point<f64>`.
The argument is a value in the unit interval `[0, 1]`.
When called with `0`, you get the start point of the line.
When called with `1`, you get the end point of the line.

Instead of declaring a `Circle` type, a `Line` type and so on,
one can use `Point<f64>` to represent both of them.

Higher order data structures makes easier to write generic algorithms for geometry.
Although it seems abstract at first, it is also practically useful in unexpected cases.

For example, an animated point can be thought of as having the type `Point<(&[Frame], f64)>`.
The first argument contains the animation data and the second argument is time in seconds.
Properties `x`, `y` and `z` of an animated point determines how the animated point is computed.
The details of the implementation can be hidden from the algorithm that uses animated points.

Sometimes you need to work with complex geometry.
In these cases, it is easier to work with higher order data structures.

For example, a planet might have a center, equator, poles, surface etc.
A planet orbits around a star, which orbits around the center of a galaxy.
This means that the properties of a planet, viewed from different reference frames,
are functions of the arguments that determine the reference frame.
You can create a "higher order planet" to reason about a planet's properties
under various reference frames.

### Design

Here is an example of how to declare a new higher order data structure:

```rust
use higher_order_core::{Ho, Call, Arg, Fun, Func};
use std::sync::Arc;

/// Higher order 3D point.
#[derive(Clone)]
pub struct Point<T = ()> where f64: Ho<T> {
    /// Function for x-coordinates.
    pub x: Fun<T, f64>,
    /// Function for y-coordinates.
    pub y: Fun<T, f64>,
    /// Function for z-coordinates.
    pub z: Fun<T, f64>,
}

// It is common to declare a type alias for functions, e.g:
pub type PointFunc<T> = Point<Arg<T>>;

// Implement `Ho<Arg<T>>` to allow higher order data structures
// using properties `Fun<T, Point>` (`<Point as Ho<T>>::Fun`).
impl<T: Clone> Ho<Arg<T>> for Point {
   type Fun = PointFunc<T>;
}

// Implement `Call<T>` to allow higher order calls.
impl<T: Copy> Call<T> for Point
    where f64: Ho<Arg<T>> + Call<T>
{
    fn call(f: &Self::Fun, val: T) -> Point {
        Point::<()> {
            x: <f64 as Call<T>>::call(&f.x, val),
            y: <f64 as Call<T>>::call(&f.y, val),
            z: <f64 as Call<T>>::call(&f.z, val),
        }
    }
}

impl<T> PointFunc<T> {
    /// Helper method for calling value.
   pub fn call(&self, val: T) -> Point where T: Copy {
       <Point as Call<T>>::call(self, val)
   }
}

// Operations are usually defined as simple traits.
// They look exactly the same as for normal generic programming.
/// Dot operator.
pub trait Dot<Rhs = Self> {
    /// The output type.
    type Output;

    /// Returns the dot product.
    fn dot(self, other: Rhs) -> Self::Output;
}

// Implement operator once for the ordinary case.
impl Dot for Point {
    type Output = f64;
    fn dot(self, other: Self) -> f64 {
        self.x * other.x +
        self.y * other.y +
        self.z * other.z
    }
}

// Implement operator once for the higher order case.
impl<T: 'static + Copy> Dot for PointFunc<T> {
    type Output = Func<T, f64>;
    fn dot(self, other: Self) -> Func<T, f64> {
        let ax = self.x;
        let ay = self.y;
        let az = self.z;
        let bx = other.x;
        let by = other.y;
        let bz = other.z;
        Arc::new(move |a| ax(a) * bx(a) + ay(a) * by(a) + az(a) * bz(a))
    }
}
```

To disambiguate impls of e.g. `Point<()>` from `Point<T>`,
an argument type `Arg<T>` is used for point functions: `Point<Arg<T>>`.

For every higher order type `U` and and argument type `T`,
there is an associated function type `T -> U`.

For primitive types, e.g. `f64`, the function type is `Func<T, f64>`.

For higher order structs, e.g. `X<()>`, the function type is `X<Arg<T>>`.

The code for operators on higher order data structures must be written twice:

- Once for the ordinary case `X<()>`
- Once for the higher order case `X<Arg<T>>`
