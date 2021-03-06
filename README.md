# Change Manager
Change Manager is a simple all-in-one system for starting, storing, and comparing changes to ActiveRecord objects using Resque and resque-scheduler. It also features and supports rspec. All tests in this gem are written with rspec. 

While Resque itself is pretty simple to implement, this packages Resque, resque-scheduler together with some simple common sense logic for comparing objects. I wrote this to simplify some functionality in my own applications, so some of it is application-specific. However, I tried to construct this gem in a way that would make customization easy. ___I welcome anyone and everyone that happens to find this useful to add your own customizations to it. All you need to do is fork the repo, make your changes, and create a Pull Request.___ Understandably, I would like any customizations made to have matching specs.

## Run-down of how ChangeManager works:
I don't like complicated code. I don't think anyone does. Therefore, ChangeManager is written with a 'no nonsense' style. First, you must create a `Change` object using `ChangeManager::Manager.queue_change()` like so:
(app/services/change_manager/manager.rb)

```
ChangeManager::Manager.queue_change(owner, change_type, context, target)
```

This is the only line you need to write to begin queueing notifications and using the gem. However the email format is purposefully generic and you may want to substitute your own views for it.

### Properties:

`owner` - string. This value is arbitrary and is generally an email address, though you can use a `User` object if you create a custom Notification model (or modify the existing model). `owner` is the entity making the change to `target`

`change_type` - string. This value has to be one of the predefined change types located in `config/change_types.yaml`. You can add your own custom change types very easily as long as each change's values are defined in a dictionary of dictionaries (I'll go over this more soon.)

`context` - string. This value is also arbitrary and can be anything you want. It is intended to be used for an id of some Model. It's mostly application specific so that's about as specific as it gets. You can also pass it an empty string - context is not a required argument.

`target` - string. This value is arbitrary and similar to `owner`: Can be whatever you want, but generally an email address or some sort of identifier. `target` is the entity on the receiving end of the change made by `owner`.

___Real-world example:___

`ChangeManager::Manager.queue_change('kyle', 'added_as_editor', 'some_random_id', 'chris')`

Generates a `Change` object with the attributes:

```
owner: 'kyle' 
change_type: 'added_as_editor' 
context: 'some_random_id' 
target: 'chris'
cancelled: cancelled #all notifications are not cancelled by default.
```

This works by calling the actual constructor for the `Change` object and then scheduling the `BeginChange` job to be queued 15 minutes from the current date/time:

```
Resque.enqueue_in(15.minutes, BeginChange, change_id)

...

#begin_change.rb:
def self.perform(change_id)
    ChangeManager::Manager.process_change(change_id)
end
```

All this job does is call the `process_change` method and passing it the id of the `Change` object from before. The `process_change` method does an initial check to see if the `Change` was cancelled. If it is cancelled, the flow ends there, nothing else happens. If it was not, it passes the owner and target of the change to `process_changes` which is just a wrapper for two different methods: `group_similar_changes` and `cancel_all_changes`.

`group_similar_changes` collects changes in an array from the database where the `owner` and `target` are the same as the initial change and where `cancelled` is false.

`similar_changes = Change.where(owner: owner, target: target, cancelled: false)`

If there are no other changes that match this criteria, the array is returned back up to `cancel_all_changes` which cancels the change, and sends it back up the stack, eventually leading to an email with a nicely formatted table to `target` (hence why target is generally an email address). If there are similar changes to be found, it passes it to another method in the Manager `cancel_inverse_changes`.

`cancel_inverse_changes` simply takes the array of similar changes and uses a nested loop to compare any two values in the array for inverse changes. Inverse changes are changes that undo each other in the context of my applications. It uses the comparison method `change.inverse_of?(next_change)` which returns true if the first change's change type is contained in the next change's `inverse:` dictionary.

If `change.inverse_of?(next_change)` returns true, it cancels both changes, then loops through the array one more time to remove them. In this way, if you have three or more inverse changes of the same type (i.e. owner adds target as a delegate, then removes target, then adds target again) it only cancels n - 1 chagnes and an email is still sent out.

After `group_similar_changes` and `inverse_of?` vet out any inverse changes, it then returns back up the stack to `cancel_all_changes` which cancels every change in the array and returns back to `process_changes` and so on to `process_change`. `process_change` then sends the array to the `notify` method whic grabs the `Change`'s from it, formats it into an action mailer, and sends it out. The `Change`'s `notified` value is updated to the date and time at the moment of emil delivery.

By default (for now) the email is sent out in the form of a table.

### Specifying custom change types:

The change types are currently only needed/loaded in the `inverse_of?` method. 

Due to the way yaml works, any data type can be a change type, but the only data type that really makes sense in this context is a string - though don't let that stop you, I love repurposing code! - As long as you follow this format, your chagne types will be valid:

```
change_type_name: # should be what you're calling your change type, ex. added_as_admin or removed_as_admin
 print: change_type_name # prints the name as a string
 inverse: inverse_change_type_name # specifies the inverse change
 human_readable: Change Type Name # prints a human-readable string for use in views and mailers.
...
 ...
 ...
 ...
```

In this way, an inverse change is referenced from an array like this: `change_types[change_type_name]['inverse']`

There are two optional attributes you can use, `print:` and `human_readable:`. These are not required for a change type. `inverse:` is.

Since the application only depends on the change types being stored in a dictionary of dictionaries, you don't necessarily need to use yaml as the backend storage of the change types. As long as you can serve them in the proper format, the application doesn't care.

Additionally if for whatever reason you can't store the change types in a yaml file, you just need to customize this line in `inverse_of?`: `@change_types[possible_inverse_change.change_type]['inverse']` to however you need to reference it. Really this method can do whatever you want, as long as it returns true when you need it to.

See? Customization made easy.
