---
title: "Modeling with opaque types in Scala 3"
#custom_js: [mouse_coords]
date: 2021-02-06T00:00:00
draft: false
categories: [Scala 3]
tags: [Scala opaque, Scala extension] 
---

I like physics. The reason behind it is that physics has a beautiful and consistent type system.
Let's start with very simple scalar model, that is used in Newton mechanics. What we need here are several types like `Time`, `Acceleration`, `Velocity`, `Displacement` for kinematics. If we also deal with dynamic, we need `Mass`, `Force`, `Momentum` and `Energy`. We are also introducing  `Constant`,  that is just a number and can be used solely as multiplier (or divider, but not implemented).

You can run the whole example  [here](https://scastie.scala-lang.org/api/download/JurajBurian/ns83iWTqSYCShiKoQ1ekIg/3).

Let's continue with our model:

```scala
object types {
  import scala.annotation.targetName
  opaque type DoubleBased = Double // root type
  type Constructor[T] = scala.Conversion[Double, T] // constructor plus conversion for free

  extension[T <: DoubleBased] (x: T) {
    def asDouble: Double = x
  }
  extension[T<: DoubleBased] (x: T)(using b: Constructor[T]) { // notation with explicit usage of using 
    def +(y: T): T = b(x + y)
  }
  // Constant
  opaque type Constant <: DoubleBased = Double
  given Constant: Constructor[Constant] = (p:Double) => p

  extension[T <: DoubleBased : Constructor] (x: Constant) { // alternative notation (not using used) 
    @targetName("Constant*")
    def *(y: T): T = summon[Constructor[T]](x * y) // summon is used to obtain Constructor
  }
  // TIME
  opaque type Time <: DoubleBased = Double
  given Time: Constructor[Time] = (p:Double) => p
  //ACCELERATION
  opaque type Acceleration <: DoubleBased = Double
  given Acceleration:Constructor[Acceleration] = (p:Double) => p

  extension (x: Acceleration) {
    @targetName("Acceleration*")
    def *(y: Time): Velocity = x * y
  }
  // VELOCITY 
  opaque type Velocity <: DoubleBased = Double
  given Velocity: Constructor[Velocity] = (p:Double) => p

  extension (x: Velocity) {
    @targetName("Velocity*")
    def *(y: Time): Displacement = x * y
  }
  // DISPLACEMENT 
  opaque type Displacement <: DoubleBased = Double
  given Displacement:Constructor[Displacement] = (p:Double) => p
  // MASS 
  opaque type Mass <: DoubleBased = Double
  given Mass:Constructor[Mass] = (p:Double) => p

  extension(x: Mass) {
    @targetName("Mass1*")
    def * (y:Velocity):Momentum = x * y
    @targetName("Mass2*")
    def * (y:Acceleration):Force= x * y
  }
  // FORCE 
  opaque type Force <: DoubleBased = Double
  given Force:Constructor[Force] = (p:Double) => p

  extension(x: Force) {
    @targetName("Force*")
    def * (y:Time):Momentum = x * y
  }
  // MOMENTUM 
  opaque type Momentum <: DoubleBased = Double
  given Momentum:Constructor[Momentum] = (p:Double) => p

  extension(x: Momentum) {
    @targetName("Momentum*")
    def * (y:Velocity):Energy = x * y
  }
  // ENERGY 
  opaque type Energy <: DoubleBased = Double
  given Energy:Constructor[Energy] = (p:Double) => p
}
```
Only multiplication and sum operators are defined in this example, but extension with `/` or `-` is pretty straightforward.
We introduced root type named `DoubleBased`, the reason for it is, that we want to have any subtype convertible to the `Double` (see `asDouble` extension ) and also what is more important, we want to have ability to use sum on any defined type (see `+` extension).  `asDouble` extension is trivial, but following `+` extension is more complicated, because it uses  `Constructor` as implicit parameter.
All constructors are declared with `given` keyword. It means, that  [givens](http://dotty.epfl.ch/docs/reference/contextual/givens.html) can be injected implicitly. See `(using b: Constructor[T])`  or `summon[Constructor[T]]` in  the above code snippet.
Importantly, the constructor function is also declared as conversion function  (`scala.Conversion`). More details about implicit conversions can be found  [here](https://dotty.epfl.ch/docs/reference/contextual/conversions.html). The consequence is, that if  `import scala.language.implicitConversions` than constructor functions work also as implicit conversions.

Let's spend some time with  `Constant: *`  extension. Here we have the same situation, where we need to use  constructor.
The reason for it is, that we want to write formulas like `Constant(10)*v` (remark:  `const * T ~ T` , meaning that type is conserved).

Let's construct some formulas:


```scala
package formulas {
  import types.{given, _}
  import scala.language.implicitConversions
  
  def velocity(u:Velocity, a:Acceleration, t:Time): Velocity = 
  	u + a*t 
  def displacement(u:Velocity, a:Acceleration, t:Time): Displacement = 
  	u*t + Constant(0.5)*a*t*t
  def energy(e0:Energy, m:Mass, v:Velocity):Energy =
  	e0 + 0.5*m*v*v
  def energy(using e0:Energy, m:Mass, a:Acceleration, t:Time):Energy = 
  	e0 + 0.5*m*a*t*(a*t) // fomula need help with brackets 
}
```

Our first formula (`velocity`) is the most simple one.
The result of  `a*t` is velocity. In this case operator `*` is from `Acceleration` extension.  Than  the `+` operator calculates the sum of two velocities.
In second formula is `Constant` constructed with constructor explicitly, in other two is implicit conversion used.

Let's put everything together into a running example:

```scala
object Main {
  def main(args: Array[String]): Unit = {
    import types.{given, _}
    import formulas._
    import scala.language.implicitConversions // try comment

    val v1 = Velocity(13.0) // direct declaration 
    val v2 = velocity(10.0, 10.0, 1.0) + v1 // sum, velocity constructed with implicit conversion
    val v3 =  v1 + (42 - 13) // implicit conversion
    println(s"v1 = $v1, v2= $v2, v3 = $v3")
    
    //val v4 =  13.5 + v3 // not compilable
       
    val e = energy(0.0, 20.0, math.sqrt(4.2)) // formula with speed, implicit conversion
    println(s"e = ${e}")
    
    given tg:Time = Time(1)
    given mg:Mass = Mass(2 * 41)
    given ag:Acceleration = Acceleration(1)
    given e0g:Energy = Energy(1)
    
    val e1 = energy // attributes provided implicitly 
    println(s"e1: ${e1}")
  }
}   
```

In our system compiler guards possible operations. For example `energy` can't be summed with `velocity` , or `velocity` multiplied by `velocity`.
We also get  implicit conversions for free, if we wanted to, just by using: `import scala.language.implicitConversions`.