# LF-Tag limitations

Please see Lake Formations public documentation for the current limitations as they may change. Many of the limits associated with LF-Tags are soft limits so if you need to increase size limits, please ensure a support case.

## Limitation #1 - You can only attach a single value of an LF-Tag tag to a resource

Suppose you want to create an LF-Tag, *AccessibleRoles* that contains a list of roles, *Engineering, Sales, Marketing* that can access a resource. If you need multiple roles to access a single resource, then you would need to specify multiple values to the resource. This is currently not supported today.

The solution here is to create multiple LF-Tags that represents each value. From the above example, we can create LF-Tags *AccessibleRoles:Engineering*, *AccessibleRoles:Sales*, and *AccessibleRoles:Marketing* with each with a single value of *true*. You can then tag a resource with appropriate LF-Tags.

## Limitation #2 - No OR expression in grant expression for multiple LF-Tags

Suppose we take the example in limitation #1, where we have LF-Tags *AccessibleRoles:Engineering*, *AccessibleRoles:Sales*, and *AccessibleRoles:Marketing*, and you want to grant a principal all resources that are tagged with *AccessibleRoles:Sales*, and *AccessibleRoles:Marketing*. LF-Tag expressions do not support OR operators between LF-Tags, ie grant user User1 to all resources tagged with *AccessibleRoles:Engineering = true OR AccessibleRoles:Sales = true". This is currently not supported with LF-Tags.

The solution here is to perform multiple grant operations, one for each LF-Tag. For example, we can have one grant expression to be *AccessibleRoles:Sales = true* , and another being *AccessibleRoles:Marketing = true*.

## Limitation #3 - No Less than operator

Some customers may want to create an LF-Tag that has some scale or hierarchy and when a grant occurs at one level, then the user should get permissions at that level and everything below it. Since LF-Tag expressions support OR operator for LF-Tag values, you can mimic the behavor. For example, if you want to create a *SensitivityLevel* tag with values *1,2,3,4,5*, if you want to grant a user resources that have *SensitivityLevel* = 3, then your grant expression can be *Sensitivity Level = 1 OR 2 OR 3*
