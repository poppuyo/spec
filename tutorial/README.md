# Tell me about the power of ops

## Step 0: Give the op a place to live
Right now, ops need a `collection.yml` to live with, so let's make one up:
```yaml
name: sample ops
description: ops to demonstrate functionality
```
For this example, I've stored this file at `~/scribbles/collection.yml`
Let's make a folder at the same level with the name of the op: `happy-birthday-op`

## Step 1: Let's go make an op

Here's a simple op definition that doesn't do anything at all that we can put into:
```yaml
name: happy-birthday-op
description: this is an op that wishes a happy birthday
```

Go ahead and save that to `~/scribbles/happy-birthday-op/op.yml`

Let's give it some inputs:
```yaml
name: happy-birthday-op
description: this is an op that wishes a happy birthday 
inputs:
  name:
    string: 
      description: your name
  age: 
    number: {}
```
We've done two things here:
1. defined `name` to be a `string` with a particularly helpful description
2. defined `age` to be a `number` and not put in any helpful description
> note: `number: {}` is used because just `number:` would be interpreted as `null`

That's great and all, but what if I had some really cool (read: simple) script that would take a name and an age and write you a greeting?
```sh
#!/bin/sh
echo happy $AGE-th birthday, $NAME
```
Let's go ahead and put that at `~/scribbles/happy-birthday-op/birthday.sh`, we'll be needing it soon.

So soon, in fact... we need it right now!

```yaml
name: happy-birthday-op
description: this is an op that wishes a happy birthday 
inputs:
  name:
    string: 
      description: your name
  age: 
    number: {}
run:
  serial:
    - container:
        image: { ref: alpine }
        cmd: [ /birthday.sh ]
        envVars:
          AGE: $(age)
          NAME: $(name)
        files: { /birthday.sh }
```
We've done something more interesting here, adding a `run` object.
Let's drill down into what ~~havoc we've introduced~~ we've just added.
* `serial` (as opposed to `parallel`) allows us to specify that whatever we've put under the `run` is done sequentially. 
> note: this isn't important now, but will be once you have more complicated `run` objects.
* `container` allows us to specify a container (`image`) to run some command (`cmd`)
* `envVars` are then provided to the running container alongside
* `files` that we want to be able to execute within the container

Running this would then provide the output below if I provided `26` and `jason` as inputs:
```
happy 26-th birthday, jason
```

## Step 2: Let's go make this op complicated

### Constraints

Extending our contrived example, let's go ahead and assume that we think names are only alphabetic in nature.
We can add constraints to enforce expectations! 
Let's look at how the `name` input would change:

```yaml
inputs:
  name:
    string: 
      description: your name
```

With a requirement of not having a null name:

```yaml
inputs:
  name:
    string: 
      description: your name
      constraints: { minLength: 1 }
```

With a requirement of having a non-null name that's also alphabetic

```yaml
inputs:
  name:
    string: 
      description: your name
      constraints: { minLength: 1, pattern: '^[a-zA-Z]+' }
```

Let's also go make this birthday op only for adults as well:

```yaml
  age: 
    number: {}
```

18+ only? let's try that

```yaml
  age: 
    number:
      constraints: { minimum: 18 }
```

At this point, I've realized that I've made a grave mistake by assuming that -th would suffice for any given age.
To (somewhat incorrectly) fix that, I'm adding a new string input.

```yaml
  ordinal-suffix:
    string:
      description: st, nd, rd, th?
      constraints: { enum: [ st, nd, rd, th ] }
```

I'll need to add a new envVar to handle that too:
```yaml
        envVars:
          AGE: $(age)
          NAME: $(name)
          SUFFIX: $(ordinal-suffix)
```

And update the script to handle the new string
```sh
#!/bin/sh
echo happy $AGE$SUFFIX birthday, $NAME
```

Putting in the following args now gives the following results:

inputs:
* age=22
* name=jason
* ordinal-suffix=nd

output:
```
happy 22nd birthday, jason
```
