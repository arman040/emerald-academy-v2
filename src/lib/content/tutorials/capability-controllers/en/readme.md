---
layout: examples
---

The old Cadence capability system is being deprecated. Namely, `link` and `getCapability` are not what you should use anymore to create capabilities.

This tutorial will walk through the new mechanism: **Capability Controllers** ("Cap Cons").

# Example Overview

In order to showcase the new Capability Controllers system, let's create at an example contract so we can compare how you would create, manage, and get the old vs. new capabilities side by side.

```cadence
pub contract HelloWorld {

    pub resource interface IGreeting {
        pub var greeting: String
    }

    pub resource Greeting: IGreeting {
        pub var greeting: String

        pub fun changeGreeting(newGreeting: String) {
            self.greeting = newGreeting
        }

        init(greeting: String) {
            self.greeting = greeting
        }
    }

    pub fun createGreeting(greeting: String): @Greeting {
        return <- create Greeting(greeting: greeting)
    }

}
```

The idea is that we have a `Greeting` resource. We want the public to be able to read the `greeting` variable, but only the owner of the resource should be able to call the `changeGreeting` function.

## Old vs. New: Creating a Public Capability

First we want to store the greeting resource in a user's storage, and then allow the public to read the greeting variable.

Here is the old way of doing that:
```cadence
import HelloWorld from 0x01

transaction(greeting: String) {
    prepare(signer: AuthAccount) {
        // store the resource
        signer.save(<- HelloWorld.createGreeting(greeting: greeting), to: /storage/MyGreeting)
        // create a public capability restricted to `IGreeting` that is linked to a storage path
        signer.link<&HelloWorld.Greeting{HelloWorld.IGreeting}>(/public/MyGreeting, target: /storage/MyGreeting)
    }
}
```

Here is the new way of doing that:
```cadence
import HelloWorld from 0x01

transaction(greeting: String) {
    prepare(signer: AuthAccount) {
        // store the resource
        signer.save(<- HelloWorld.createGreeting(greeting: greeting), to: /storage/MyGreeting)
        // create a storage capability, previously known as a "private capability"
        let cap = signer.capabilities.storage.issue<&HelloWorld.Greeting{HelloWorld.IGreeting}>(/storage/MyGreeting)
        // publish the storage ("private") capability to a public path
        signer.capabilities.publish(cap, /public/MyGreeting)
    }
}
```

As you can see, instead of just linking a certain type to the public from a storage path, it is now broken up into 2 steps:
1. Create a new storage capability (previously known as a "private capability") first
2. Issue that storage capability to a public path

## Old vs. New: Getting a Public Capability

Now let's walk through how you would actually get the capability from a user's account.

Here is the old way of doing that:
```cadence
import HelloWorld from 0x01

pub fun main(user: Address): String {
    let greetingCap: Capability<&HelloWorld.Greeting{HelloWorld.IGreeting}> = getAccount(user).getCapability<&HelloWorld.Greeting{HelloWorld.IGreeting}>(/public/MyGreeting)
    let greetingRef: &HelloWorld.Greeting{HelloWorld.IGreeting} = greetingCap.borrow() ?? panic("This public capability does not exist.")

    return greetingRef.greeting
}
```

Here is the new way of doing that:
```cadence
import HelloWorld from 0x01

pub fun main(user: Address): String {
    // Note how the get function returns an optional now.
    let greetingCap: Capability<&HelloWorld.Greeting{HelloWorld.IGreeting}> = getAccount(user).capabilities.get<&HelloWorld.Greeting{HelloWorld.IGreeting}>(/public/MyGreeting)
                                                                                    ?? panic("This public capability does not exist.")
    let greetingRef: &HelloWorld.Greeting{HelloWorld.IGreeting} = greetingCap.borrow()!

    return greetingRef.greeting
}
```

OR, you can use a more simplified version using the new `borrow` conveniance function:
```cadence
import HelloWorld from 0x01

pub fun main(user: Address): String {
    let greetingRef: &HelloWorld.Greeting{HelloWorld.IGreeting} = getAccount(user).capabilities.borrow<&HelloWorld.Greeting{HelloWorld.IGreeting}>(/public/MyGreeting)
                                                                        ?? panic("This public capability does not exist.")

    return greetingRef.greeting
}
```

## Old vs. New: Removing a Public Capability

Here is the old way of doing that:
```cadence
import HelloWorld from 0x01

transaction(greeting: String) {
    prepare(signer: AuthAccount) {
        signer.unlink(/public/MyGreeting)
    }
}
```

Here is the new way of doing that:
```cadence
import HelloWorld from 0x01

transaction(greeting: String) {
    prepare(signer: AuthAccount) {
        signer.capabilities.unpublish(/public/MyGreeting)
    }
}
```

## Old vs. New: Creating & Getting a Storage ("Private") Capability

As you've seen above, public capabilities are pretty similar. It is just different syntax.

However, private capabilities are very different now. 

Here is the old way of creating a private capability:
```cadence
import HelloWorld from 0x01

transaction(greeting: String) {
    prepare(signer: AuthAccount) {
        // store the resource
        signer.save(<- HelloWorld.createGreeting(greeting: greeting), to: /storage/MyGreeting)
        // create a private capability that is linked to a storage path
        signer.link<&HelloWorld.Greeting>(/private/MyGreeting, target: /storage/MyGreeting)


        // then, at a later point (potentially in a different transaction), you would borrow it and distribute it like this:
        let privateCap: Capability<&HelloWorld.Greeting> = signer.getCapability<&HelloWorld.Greeting>(/private/MyGreeting)
        // ... and you would then move it around as you please
    }
}
```

Here is the new way of creating a private capability:
```cadence
import HelloWorld from 0x01

transaction(greeting: String) {
    prepare(signer: AuthAccount) {
        // store the resource
        signer.save(<- HelloWorld.createGreeting(greeting: greeting), to: /storage/MyGreeting)
        // create a storage capability, previously known as a "private capability"
        let privateCap = signer.capabilities.storage.issue<&HelloWorld.Greeting>(/storage/MyGreeting)
        // ... and you would then move it around as you please
    }
}
```

What's really important to see here is that in the old system, you would just create a private capability one time and then be able to get that one capability whenever you'd like. In the new system, every time you want to get the storage ("private") capability, you have to make a new one. There is no way to go back and get an old private capability you created.

The downside here is that you waste a lot of storage by having to fetch a private capability, because you're not really fetching it, you're making a new one. The benefit is that you can now individually control the private capabilities you're giving out. So unlike the old way where unlinking a private capability takes away control from *everyone* who has it, in the new way you can individually unlink certain capabilities.

Also note that private paths do not exist anymore. You are instead just issuing storage capabilities on a certain storage path. Because of this, you need a way to view all the capabilities you've created for a certain storage path so you can individually manage them. Here is how you'd do that:
```cadence
import HelloWorld from 0x01

pub fun main(user: Address, storagePath: StoragePath): [&AuthAccount.StorageCapabilityController] {
    let authAccount: AuthAccount = getAuthAccount(user)
    return authAccount.capabilities.storage.getControllers(forPath: storagePath)
}
```

Here is what the `&AuthAccount.StorageCapabilityController` type looks like:

```cadence
pub struct StorageCapabilityController {

    /// An arbitrary "tag" for the controller.
    /// For example, it could be used to describe the purpose of the capability.
    /// Empty by default.
    pub var tag: String

    /// The type of the controlled capability, i.e. the T in `Capability<T>`.
    pub let borrowType: Type

    /// The identifier of the controlled capability.
    /// All copies of a capability have the same ID.
    pub let capabilityID: UInt64

    /// Delete this capability controller,
    /// and disable the controlled capability and its copies.
    ///
    /// The controller will be deleted from storage,
    /// but the controlled capability and its copies remain.
    ///
    /// Once this function returns, the controller is no longer usable,
    /// all further operations on the controller will panic.
    ///
    /// Borrowing from the controlled capability or its copies will return nil.
    ///
    pub fun delete()

    /// Returns the targeted storage path of the controlled capability.
    pub fun target(): StoragePath

    /// Retarget the controlled capability to the given storage path.
    /// The path may be different or the same as the current path.
    pub fun retarget(_ target: StoragePath)
}
```

The most important thing to note here is the `capabilityID`. Each capability now has an ID that you need to know to manage it.

## Old vs. New: Removing a Storage ("Private") Capability

Here is the old way of doing that:
```cadence
import HelloWorld from 0x01

transaction(greeting: String) {
    prepare(signer: AuthAccount) {
        signer.unlink(/private/MyGreeting)
    }
}
```

Here is the new way of doing that:
```cadence
import HelloWorld from 0x01

transaction(greeting: String, capabilityID: UInt64) {
    prepare(signer: AuthAccount) {
        let capController: &AuthAccount.StorageCapabilityController = signer.capabilities.storage.getController(byCapabilityID: capabilityID)
                                                                            ?? panic("This Cap Con does not exist.")
        capController.delete()
    }
}
```

In the new way, you must know the `capabilityID`. Unfortunately this means that you as the developer will have to keep track of which ID belongs to which people to know how which one to unlink. Remember, you can also view all the capability IDs using the code in the above section.

## Conclusion

There are other new features included in the new CapCon system, but I wanted this to outline the exact differences between the old Capabilities and new CapCon api. I hope this helped!

Also, if you'd like to view the official proposal for Cap Cons, go <a href="https://github.com/onflow/flips/blob/f7d2b5c99dbed3821c2662f28b06f6646b5eb3ee/cadence/20220203-capability-controllers.md">here</a>.

Till next time ~ Jacob Tucker