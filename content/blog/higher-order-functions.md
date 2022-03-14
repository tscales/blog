+++
title = "A Practical Use Case for Higher Order Functions"
date = "2022-03-13T11:02:32-07:00"

#
# description is optional
#
# description = "An optional description for SEO. If not provided, an automatically created summary will be used."

tags = ["functional-programming"]
+++

Have you ever encountered this scenario before?

Lets say you have a class `User`.

```scala
case class User(firstName: String, lastName: String, address: String)
```
You're using this class in production to store information retrieved from an endpoint. Now imagine you are given a new task where you need to fetch a User from a new endpoint. However, This endpoint _does not contain the address field_, you'll have to get that field from a second new endpoint. What do you do?

One solution could be to create 2 new classes and a function that combines them in to a user. 

```scala
case class UserWithoutAddress(firstName: String, lastName: String)
case class UserAddress(address: String)

def makeUser(partialUser: UserWithoutAddress, address: UserAddress): User = 
    User(partialUser.firstName, partialUser.lastName, address.address)
```  

What if instead of 3 fields in your `User` class, you have 20?

With a higher order function, these extra classes can be avoided all together. Instead of a returning a `UserWithoutAddress`, we can return a function `String => User` that will complete the User for us once we obtain the address at a later time.

```scala
def getUserWithoutAddress(id: String): String => User = {
    User("foo", "bar", _)
}

val partialUser: String => User = getUserWithoutAddress(1)
val user: User = partialUser("123 main st")
```

If you use Circe, you can even define your json decoders as such:

```scala
implicit val partialUser: Decoder[String => User] = {
    (c: Hcursor) => 
        for {
            firstName <- c.downField("firstName").as[String]
            lastName <- c.downField("lastName").as[String]
        } yield User(firstName, lastName, _)
}
```



