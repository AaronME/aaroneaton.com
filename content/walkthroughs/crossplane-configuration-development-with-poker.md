---
title: Teaching Crossplane to Play Poker
date: 2022-03-10
description: 'Learn the Crossplane Resource Model by Teaching Crossplane to Play Poker'
image: images/crossplane/crossplane-configuration-development-with-poker/poker-header.png
aliases:
- /crossplane/crossplane-configuration-development-with-poker/
---

## Let's Teach Crossplane to Play Poker

The [Penguin Book of Card Games](https://www.amazon.com/dp/B002XHNNX4) lists over 250 games which can be played with a standard deck of playing cards. Most of those games don't require any changes to the deck. Blackjack, poker, gin, go fish, crazy eights, old maid... these games all use the same 4 suits of 13 ranks.

When we start composing infrastructure with Crossplane, it's a lot like learning a new game with playing cards. We know the pieces. We know what they do. What we need to learn is how we're going to arrange them to play a new game.

In this demo, I'm going to walk through an example Crossplane Configuration
Package Development Cycle. Along the way I'm going to use some of [the tools](https://aaroneaton.com/crossplane/crossplane-package-testing-with-kuttl/)
I've talked about in previous posts. And I'm going to demystify some of the
concepts around Crossplane Composition.

I'm going to do this by teaching Crossplane to Play Poker.

## Let's get Started!

You can find a complete example of this tutorial in the [A Tour of
Crossplane](https://github.com/AaronME/ATourOfCrossplane/tree/main/crossplane-configuration-development-with-poker)
repo.

### Prerequisites

We're going to assume you have the following installed:
- [kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)
- [up](https://cloud.upbound.io/docs/getting-started/install-and-setup)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [helm](https://helm.sh/docs/intro/install/)
- [kuttl](https://kuttl.dev)

### Provider-Cards

To play a game, we need cards. We built a custom provider,
[provider-cards](https://github.com/aaronme/provider-cards), which will give us
a complete deck of standard playing cards to work with.

We will install the provider when we install crossplane with a
__uxp-values.yaml__ file. We store this file under __tests/__ so it can be
re-used in our kuttl test bootstrapping:

__tests/uxp-values.yaml__:
```yaml
provider:
  packages:
    - registry.upbound.io/aaroneaton/provider-cards:v0.0.1
```

### Set up a Casino

Launch a kind cluster and install crossplane:

```bash
kind create cluster --name cards
up uxp install -f tests/uxp-values.yaml
```

### Open a New Deck and Give it a Shuffle

We open a new Deck by creating a ProviderConfig, and we shuffle it with a
Secret. Every ProviderConfig represents a new set of 52 playing cards. The
provider will only issue one card of each type (say, the ♥10) for each
ProviderConfig.

We aren't going into too much detail about Providers here. If you are looking
for more information, take a look at the [crossplane
docs](https://crossplane.io/docs/v1.6/concepts/providers.html). 

Apply the following manifests to get a new deck:

__tests/init.yaml__:
```yaml
---
apiVersion: v1
kind: Secret
metadata:
  namespace: upbound-system
  name: tour
type: Opaque
stringData:
  credentials: |
    { "seed": 1 }
---
apiVersion: providercards.aaroneaton.com/v1alpha1
kind: ProviderConfig
metadata:
  name: tour
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: upbound-system
      name: tour
      key: credentials
```
And apply it to your cluster:

```bash
kubectl apply -f tests/init.yaml
```

Now we can play with our cards.

### Lesson One: Cards are Managed Resources

![Cards are Managed Resources](../../images/crossplane/crossplane-configuration-development-with-poker/managed-resource.png "Cards are Managed Resources")

Crossplane is built to operate Managed Resources. Today, the Managed Resources
we are going to use are Cards.

Let's pull one card off the top of the deck:

```yaml
---
apiVersion: providercards.aaroneaton.com/v1alpha1
kind: Card
metadata:
  name: acard
spec:
  forProvider: {}
  providerConfigRef:
    name: tour
```

You can check the value of __acard__ by running the following:

```bash
kubectl get card acard -o jsonpath='{@.status.atProvider.face}'        
```

You should see __♥10__ in your output. Because we declared the seed, we can
predict the output. (Golang lists do not guarantee order, though, so if I'm
wrong... sorry.)

You can deal more cards and see how they each behave. When you're ready, put all
the cards back in the deck with:

```bash
kind delete cluster --name cards
```

## What Makes a Crossplane Composition

Whether you're playing Poker, or Blackjack, or Solitaire the actual cards do not
change. What changes is how the cards are moved around the table. How many cards
are dealt to each player, which cards are showing, which cards are shared on the
table. Is there a discard pile? Do we need to "pass" cards (as in Hearts)?

These are the rules of the Game. When we understand and agree on the rules of
the game, we can build tools that make playing the game easier and more fun.

Let's make sure we all know and agree on the game we are playing.

### The Rules of Poker (Texas Hold'em Style)

Texas Hold'em is played in four rounds. In each round, we are going to deliver new
cards to the table.

This is what a game of Texas Hold'em looks like:

1. Players sit at the table and request __a hand__. __A Hand__ is made up of two
   cards. Players can only see their own cards (no cards are face up).
1. Once all players have their hand, the dealer will deal __the flop__. __The
   Flop__ is made up of one burned card (not shown to anyone at the table) and
   three face-up cards on the table. All players can view the faces of __the
   flop__.
1. After a round of betting, the dealer will deal __the turn__. __The Turn__ is
   made up of one burned card and one face-up card. Again, all players can see
   the face of __the turn__. 
1. After another round of betting, the dealer will deal __the river__. __The
   River__ is made up of one burned card and one face-up card. All players can
   see the face of __the river__.

In the steps above I have highlights all of the objects which are made out of
cards. These objects represent Composite Resources, which we will build in the
next section.

The Composite Resources we are going to build are:
- A Player Hand (2 private cards for each player)
- The Flop (1 burned card, 3 shared cards)
- The Turn (1 burned card, 1 shared card)
- The River (1 burned card, 1 shared card)

### How do we bet?

We do not bet with cards. In order to implement Composites for betting, we would
need __provider-chips__ or __provider-toothpicks__. For now, we'll assume our
players are betting by some other means.

### What about scoring?

We do not score with cards, either. In this demo, we are assuming everyone is
familiar with the rules of poker, and we want to make it faster and easier for
them to play it.

Whether a straight-flush beats a full-house is something the players will need
to agree on.

Let's start building our game.

## A PlayerHand is a Composite Resource 

![PlayerHands are Composite Resources](../../images/crossplane/crossplane-configuration-development-with-poker/composite-hand.png "PlayerHands are Composite Resources")

A Composite Resource is container for multiple other Resources. It can contain
Managed Resources from a provider, managed resources from non-Crossplane
controllers, or native Kubernetes resources (pods, configMaps, etc).

In shorthand we often refer to Composite Resources as "XRs".

### We Start by Defining the Output

The PlayerHand Composite will generate 2 private cards that only the player can
see.

The private Cards will be created at Cluster-Scope, but their Face values will
be added to a secret in the Player's Namespace.

Let's create a kuttl test which defines this output:

__tests/playerhand/00-assert.yaml__:
```yaml
apiVersion: kuttl.dev/v1beta1
kind: TestAssert
timeout: 30

---
apiVersion: providercards.aaroneaton.com/v1alpha1
kind: Card
metadata:
  name: playerone-01
spec:
  forProvider: {}
  providerConfigRef:
    name: tour

---
apiVersion: providercards.aaroneaton.com/v1alpha1
kind: Card
metadata:
  name: playerone-02
spec:
  forProvider: {}
  providerConfigRef:
    name: tour

---
apiVersion: v1
kind: Secret
metadata:
  name: my-cards
type: connection.crossplane.io/v1alpha1
```

**Note**: The Secret in our test-case does not have a Namespace. That is because
kuttl will automatically generate a namespace for the test and will only find
the secret within that same namespace.

### Define the Input

The goal of a Composite is to place many Resources behind a single, custom
resource. So we will define what that custom resource looks like as an input to
our test case.

Looking at the output, we identify some pieces of information the player needs
to supply. We see:

- 'playerone' in the names of the cards.
- 'tour' in the ProviderConfig names.
- 'my-cards' as the name of the Secret.

'Playerone' is obviously the player name. And the providerConfig creates
different decks, so let's assume that 'tour' is the name of the deck our player
wishes to play from. Let's call that the "table" the player wishes to sit at.

So we need __playerName__ and __tableName__ parameters in our XR.

The Secret is of type __type: connection.crossplane.io/v1alpha1__. This means it
was written by Crossplane. We will let players define their own secret name
using the __writeConnectionSecretToRef__ field.

Our target Composite look like this:

__tests/playerhand/00-playerhand.yaml__:
```yaml
---
apiVersion: cardgames.aaroneaton.com/v1alpha1
kind: PlayerHand
metadata:
  name: my-hand
spec:
  playerName: playerone
  tableName: tour
  writeConnectionSecretToRef:
    name: my-cards
```

Now we know our inputs and our outputs. Let's teach Crossplane how to build our
composition.

### Define the Crossplane Resource Definition

A composite is made up of two files. The first file is called the Composite
Resource Definition, or XRD for short.

If you have worked with [Custom Resource
Definitions](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)
before, you'll be familiar with Composite Resource Definitions. They define the
parameters from our input above.

Create a __package/playerhand__ folder and add this file: 

__package/playerhand/definition.yaml__:
```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  # The name must match "spec.names.plural + . + spec.group"
  name: playerhands.cardgames.aaroneaton.com
spec:
  # This group name is entirely up to you. Including your domain protects you
  # from colliding with another composition with a similar group name.
  group: cardgames.aaroneaton.com
  names:
    kind: PlayerHand # This is the Composite Resource Name
    plural: playerhands # This is the plural of the Composite Resource Name
  versions:
    - name: v1alpha1 # You can support multiple versions
      served: true
      referenceable: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties: {}
```

This is the boilerplate for our Composite. Now, let's add the parameter fields
we set in our test input:

__package/playerhand/definition.yaml__:
```yaml
...
  versions:
    - name: v1alpha1 # You can support multiple versions
      served: true
      referenceable: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                playerName:
                  type: string
                  description: Name of Player holding this Hand
                tableName:
                  type: string
                  description: Name of Table for this Hand
              required:
                - playerName
                - tableName
```

We marked both of these fields "required." This means that anyone trying to
submit a PlayerHand without those fields will get a rejection at the client
level. This is faster than watching for the error in Crossplane logs.

Notice that __versions__ is a list. You can serve multiple versions of your
Composite. This is useful if you need to add fields later. Just remember that
only one version can be the "storage" version. And it will need to support every
field that every other version might need.

### Creating Claims

Earlier we mentioned that Managed Resources were Cluster-Scoped. Well, so are
Composite Resources. In order to grant Players the right to request PlayerHands,
we must enable them as Composite Resource Claims (or XRC for short).

When a player makes a "claim," Crossplane will create a matching Composite
Resource and deliver the Secrets for that resource to the Claim's namespace.

To enable Claims, add __claimNames__ to your XRD:

__package/playerhand/definition.yaml__:
```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  # The name must match "spec.names.plural + . + spec.group"
  name: playerhands.cardgames.aaroneaton.com
spec:
  claimNames:
    kind: DealMeIn
    plural: dealmeins
  group: cardgames.aaroneaton.com
  names:
    kind: PlayerHand # This is the Composite Resource Name
    plural: playerhands # This is the plural of the Composite Resource Name
```

Wait! How can the Composite be called "PlayerHand" but the Claim is called
"DealMeIn?"

Because Claim Names do not have to have anything to do with their Composites.
This is actually a great way to help Players and Admins. Players can be given
Claim Names that are meaningful to their side of a task, while Admins get to see
Composite Names that are descriptive and meaningful to them.

Just as we used "tableName" to set the ProviderConfig value, we are using the
Claim Name to make the process _easier on the player__. Our goal is to make
playing the game easier and fun.

Let's go back and update our test to use the Claim instead of the Composite:

__tests/playerhand.00-playerhand.yaml__:
```yaml
---
apiVersion: cardgames.aaroneaton.com/v1alpha1
kind: DealMeIn
metadata:
  name: my-hand
...
```

Because the cards are being created at the cluster-scope, the only way players
will be able to see the values of their cards is via a __connectionSecret__.
This secret will exist only in the player's namespace, safe from the prying eyes
of their competitor.

### Defining ConnectionSecrets

The Players are expecting a secret with the Face Values of 2 cards in it. We
define these Secret Keys in the definition:

__package/playerhand/definition.yaml__:
```yaml
...
  names:
    kind: PlayerHand
    plural: playerhands
  connectionSecretKeys:
    - "01"
    - "02"
  versions:
...
```

Our final XRD looks like this:

__package/playerhand/definition.yaml__:
```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: playerhands.cardgames.aaroneaton.com
spec:
  claimNames:
    kind: DealMeIn
    plural: dealmeins
  group: cardgames.aaroneaton.com
  names:
    kind: PlayerHand
    plural: playerhands
  connectionSecretKeys:
    - "01"
    - "02"
  versions:
    - name: v1alpha1
      served: true
      referenceable: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                playerName:
                  type: string
                  description: Name of Player holding this Hand
                tableName:
                  type: string
                  description: Name of Table for this Hand
              required:
                - playerName
                - tableName
```

Now we have the XRD and the Test. Let's write the actual Composition!

### Composing our PlayerHand

The second half of making a Composite is the Composition.

Actually, this can be the Second, Third, Fourth, Fifth... any number of halfs.
Because you can have _multiple_ Compositions for a single Composite.

You can have a "Poker" composition for PlayerHand. Or a "Blackjack" composition.
Most of these compositions require the same input (playerName and tableName) and
not much else. So why not reUse a Composite that is working for us?

We'll cover label selectors and multiple compositions at another time. For now,
we will focus on the poker composition alone.

__package/playerhand/composition.yaml__:
```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: playerhand-composition
spec:
  # This field is necessary, the Claim will copy secrets from this namespace
  writeConnectionSecretsToNamespace: upbound-system
  compositeTypeRef:
    apiVersion: cardgames.aaroneaton.com/v1alpha1
    kind: PlayerHand
  resources: []
```

The __metadata.name__  can be anything you like. You will see many packages that
continue to use [the api group](https://github.com/upbound/platform-ref-aws/blob/a386174f05e6a99a16afe29bedb06e3e34535f78/cluster/composition.yaml#L4)
in the composition name. This is a strong convention, but is not a requirement.

Managed Resources are added to our composition as __bases__.

Let's add our first card to the list of resources. Start by just dropping in the
Resource as it is defined in our test:

__package/playerhand/composition.yaml__:
```yaml
...
  resources:
    - base:
        apiVersion: providercards.aaroneaton.com/v1alpha1
        kind: Card
        metadata:
          name: playerone-01
        spec:
          forProvider: {}
          providerConfigRef:
            name: tour
```

We want the __face__ value of the card delivered to our connection secret, so we
add those fields next:

__package/playerhand/composition.yaml__:
```yaml
...
          providerConfigRef:
            name: tour
      connectionDetails:
        - type: FromFieldPath
          name: "01"
          fromFieldPath: status.atProvider.face
```

This only works because the XRD already contains the "01" connectionSecretKey.

**Note**: provider-cards resources do not export any connection secrets on their
own. If they did, those keys would automatically be added to the Composite. But
keys copied from a status field, like above, need to be declared

Now we need to set the table name parameter the user will supply as the
providerConfigRef.name field. We do this with a patch]: 

__package/playerhand/composition.yaml__:
```yaml
...
      connectionDetails:
        - type: FromFieldPath
          name: "01"
          fromFieldPath: status.atProvider.face
      patches:
        - fromFieldPath: spec.tableName
          toFieldPath: spec.providerConfigRef.name
```

And lastly, we need to patch in the player name. To meet the requirements of the
test, we need to transform the player name into the card name by appending "-01"
to it. Crossplane has a
[function](https://crossplane.io/docs/v1.6/reference/composition.html#transform-types)
specifically for this. 

__package/playerhand/composition.yaml__:
```yaml
...
      patches:
        - fromFieldPath: spec.playerName
          transforms:
            - type: string
              string:
                fmt: "%s-01"
          toFieldPath: metadata.name
        - fromFieldPath: spec.tableName
          toFieldPath: spec.providerConfigRef.name
```

Because we have patched the metadata.name field and the providerConfigRef.name
field, we can remove them from the composition. This is good practice for
required fields which do not need a default value.

We repeat the base and patches for Card 02, and the final Composition looks like
this:

__package/playerhand/composition.yaml__:
```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: playerhand-composition
spec:
  compositeTypeRef:
    apiVersion: cardgames.aaroneaton.com/v1alpha1
    kind: PlayerHand
  resources:
    - base:
        apiVersion: providercards.aaroneaton.com/v1alpha1
        kind: Card
        spec:
          forProvider: {}
      connectionDetails:
        - type: FromFieldPath
          name: "01"
          fromFieldPath: status.atProvider.face
      patches:
        - fromFieldPath: spec.playerName
          transforms:
            - type: string
              string:
                fmt: "%s-01"
          toFieldPath: metadata.name
        - fromFieldPath: spec.tableName
          toFieldPath: spec.providerConfigRef.name
    - base:
        apiVersion: providercards.aaroneaton.com/v1alpha1
        kind: Card
        spec:
          forProvider: {}
      connectionDetails:
        - type: FromFieldPath
          name: "02"
          fromFieldPath: status.atProvider.face
      patches:
        - fromFieldPath: spec.playerName
          transforms:
            - type: string
              string:
                fmt: "%s-02"
          toFieldPath: metadata.name
        - fromFieldPath: spec.tableName
          toFieldPath: spec.providerConfigRef.name
```

### Finalize the Test

Now that our Composite Definition exists, we'll need to apply it in our test
case:

__tests/playerhand/00-playerhand.yaml__:
```yaml
apiVersion: kuttl.dev/v1beta1
kind: TestStep
commands:
  # Install XRDs
  - command: kubectl apply -f "${PWD}/package/playerhand/"
  # Wait for XRD to become "established"
  - command: kubectl wait --for condition=established --timeout=20s xrd/playerhands.cardgames.aaroneaton.com
---
apiVersion: cardgames.aaroneaton.com/v1alpha1
kind: DealMeIn
metadata:
  name: my-hand
spec:
  playerName: playerone
  tableName: tour
  writeConnectionSecretToRef:
    name: my-cards
```

If you didn't do it already, set up the Kuttl TestSuite file in the root of the
repo:

__kuttl-test.yaml__:
```yaml
apiVersion: kuttl.dev/v1beta1
kind: TestSuite
commands:
  # Install Univerasal Crossplane (uxp).
  - command: helm install universal-crossplane https://charts.upbound.io/stable/universal-crossplane-1.6.3-up.1.tgz --namespace upbound-system --wait --create-namespace --values tests/uxp-values.yaml
  # Wait for provider-cards to become healthy.
  - command: kubectl wait --for condition=healthy  --timeout=300s provider/aaroneaton-provider-cards
  # Set up a table for play
  - command: kubectl apply -f "${PWD}/tests/init.yaml"
testDirs:
  - tests/
kindContext: kuttl-test
startKIND: true
```

Try your test:
```bash
kubectl kuttl test
```

It worked! Let's add more resources!

## The Flop is _also_ A Composite Resource

![TheFlops are Composite Resources](../../images/crossplane/crossplane-configuration-development-with-poker/composite-flop.png "TheFlops are Composite Resources")

TheFlop Composite will generate 4 private cards.

One of the cards will be "burned" and not shown to anyone. The other three will
be placed "face up" -- their values will be added to a secret in a shared
namespace for all players to view. 

### Define the Output

We add a test case for the four cards and secret.

__tests/theflop/00-assert.yaml__:
```yaml
apiVersion: kuttl.dev/v1beta1
kind: TestAssert
timeout: 30

---
apiVersion: providercards.aaroneaton.com/v1alpha1
kind: Card
metadata:
  name: tour-flop-burn
spec:
  forProvider: {}
  providerConfigRef:
    name: tour

---
apiVersion: providercards.aaroneaton.com/v1alpha1
kind: Card
metadata:
  name: tour-flop-01
spec:
  forProvider: {}
  providerConfigRef:
    name: tour

---
apiVersion: providercards.aaroneaton.com/v1alpha1
kind: Card
metadata:
  name: tour-flop-02
spec:
  forProvider: {}
  providerConfigRef:
    name: tour

---
apiVersion: providercards.aaroneaton.com/v1alpha1
kind: Card
metadata:
  name: tour-flop-03
spec:
  forProvider: {}
  providerConfigRef:
    name: tour

---
apiVersion: v1
kind: Secret
metadata:
  name: flop-cards
type: connection.crossplane.io/v1alpha1
```

### Define the Input

We already know this is going to have a Claim Name, so we'll use it.

__tests/theflop/00-theflop.yaml__:
```yaml
apiVersion: kuttl.dev/v1beta1
kind: TestStep
commands:
  # Install XRDs
  - command: kubectl apply -f "${PWD}/package/theflop/"
  # Wait for XRD to become "established"
  - command: kubectl wait --for condition=established --timeout=20s xrd/theflops.cardgames.aaroneaton.com
---
apiVersion: cardgames.aaroneaton.com/v1alpha1
kind: DealTheFlop
metadata:
  name: tour
spec:
  tableName: tour
  writeConnectionSecretToRef:
    name: flop-cards
```

We don't need playerName here. The cards will be written to a shared namespace,
and a "Dealer" will need to actually create __DealTheFlop__ in that namespace.

### Isn't Provider-Cards the Dealer?

Nope, provider-cards is the deck.

### Define TheFlop XRD and Compositions

There are only a few changes to the Definition file.

Mainly, we are declaring 2 connectionSecretKeys instead of 3. (We are not adding
4, because that fourth card is "burned" and not meant to be shown):

__package/theflop/definition.yaml__:
```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: theflops.cardgames.aaroneaton.com
spec:
  claimNames:
    kind: DealTheFlop
    plural: dealtheflops
  group: cardgames.aaroneaton.com
  names:
    kind: TheFlop
    plural: theflops
  connectionSecretKeys:
    - "01"
    - "02"
    - "03"
  versions:
    - name: v1alpha1
      served: true
      referenceable: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                tableName:
                  type: string
                  description: Name of Table for this Flop
              required:
                - tableName
```

The Composition for TheFlop does not include the playerName patch, but it still
generates names based on the name of the Claim:

__package/theflop/composition.yaml__:
```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: theflop-composition
spec:
  writeConnectionSecretsToNamespace: upbound-system
  compositeTypeRef:
    apiVersion: cardgames.aaroneaton.com/v1alpha1
    kind: TheFlop
  resources:
    - base:
        apiVersion: providercards.aaroneaton.com/v1alpha1
        kind: Card
        spec:
          forProvider: {}
      # this burned card is _not_ added to the connectionSecrets
      patches:
        - fromFieldPath: spec.tableName
          transforms:
            - type: string
              string:
                fmt: "%s-flop-burn"
          toFieldPath: metadata.name
        - fromFieldPath: spec.tableName
          toFieldPath: spec.providerConfigRef.name
    - base:
        apiVersion: providercards.aaroneaton.com/v1alpha1
        kind: Card
        spec:
          forProvider: {}
      connectionDetails:
        - type: FromFieldPath
          name: "01"
          fromFieldPath: status.atProvider.face
      patches:
        - fromFieldPath: spec.tableName
          transforms:
            - type: string
              string:
                fmt: "%s-flop-01"
          toFieldPath: metadata.name
        - fromFieldPath: spec.tableName
          toFieldPath: spec.providerConfigRef.name
    - base:
        apiVersion: providercards.aaroneaton.com/v1alpha1
        kind: Card
        spec:
          forProvider: {}
      connectionDetails:
        - type: FromFieldPath
          name: "02"
          fromFieldPath: status.atProvider.face
      patches:
        - fromFieldPath: spec.tableName
          transforms:
            - type: string
              string:
                fmt: "%s-flop-02"
          toFieldPath: metadata.name
        - fromFieldPath: spec.tableName
          toFieldPath: spec.providerConfigRef.name
    - base:
        apiVersion: providercards.aaroneaton.com/v1alpha1
        kind: Card
        spec:
          forProvider: {}
      connectionDetails:
        - type: FromFieldPath
          name: "03"
          fromFieldPath: status.atProvider.face
      patches:
        - fromFieldPath: spec.tableName
          transforms:
            - type: string
              string:
                fmt: "%s-flop-03"
          toFieldPath: metadata.name
        - fromFieldPath: spec.tableName
          toFieldPath: spec.providerConfigRef.name
```

### Testing TheFlop

Run your tests again to confirm everything is working.

## The Turn and the River are _also_ Composite Resources

![TheTurns are Composite Resources](../../images/crossplane/crossplane-configuration-development-with-poker/composite-turn-river.png "TheTurns are Composite Resources") ![TheRivers are Composite Resources](../../images/crossplane/crossplane-configuration-development-with-poker/composite-turn-river.png "TheRivers are Composite Resources")

You can now make two more composites.

These are almost identical to __TheFlop__, except each plays only 1 face-up card
instead of 3. Go ahead and create your test cases, XRDs and Compositions for
__TheTurn__ (claimName "DealTheTurn") and __TheRiver__ (claimName "DealTheRiver").

Make sure your tests are passing:

```bash
kubectl kuttl test
...
--- PASS: kuttl (114.33s)
    --- PASS: kuttl/harness (0.00s)
        --- PASS: kuttl/harness/playerhand (5.87s)
        --- PASS: kuttl/harness/theriver (6.46s)
        --- PASS: kuttl/harness/theturn (7.30s)
        --- PASS: kuttl/harness/theflop (7.47s)
```

If you get the above output you're ready to play some Poker!

## This is How to Play Poker with Crossplane

Start a cluster, install __provider-cards__ and your __composites__. Remember to
provision a credential secret and your __providerConfig__.

### Round One: Deal the Hand

Create a set of users, or ask several colleagues to join you on a cluster. Make
sure each of them can only see their own namespace and the shared "table"
namespace.

Every player in the game will create a __DealMeIn__:

```yaml
---
apiVersion: cardgames.aaroneaton.com/v1alpha1
kind: DealMeIn
metadata:
  name: my-hand
  namespace: <player namespace>
spec:
  playerName: <player-name>
  tableName: <providerConfig name>
  writeConnectionSecretToRef:
    name: my-cards
```

### Round Two: Deal the Flop

Next, whoever is the dealer will create a __DealTheFlop__ in the shared
namespace:

```yaml
---
apiVersion: cardgames.aaroneaton.com/v1alpha1
kind: DealTheFlop
metadata:
  name: tour
  namespace: <shared namespace>
spec:
  tableName: <providerConfig name>
  writeConnectionSecretToRef:
    name: flop-cards
```

Players can investigate TheFlop by running:
```
kubectl get secret flop-cards -n <shared namespace> -o yaml
```

Players may bet by another means. When everyone has folded or called...

### Round Three: Deal the Turn

The dealer creates a __DealTheTurn__ in the shared namespace:

```yaml
---
apiVersion: cardgames.aaroneaton.com/v1alpha1
kind: DealTheTurn
metadata:
  name: tour
  namespace: <shared namespace>
spec:
  tableName: <providerConfig name>
  writeConnectionSecretToRef:
    name: turn-card
```

Players can investigate TheTurn by running:
```
kubectl get secret turn-card -n <shared namespace> -o yaml
```
Players bet again. When everyone has folded or called...

### Round Four: Deal the River

The dealer creates a __DealTheRiver__ in the shared namespace:

```yaml
---
apiVersion: cardgames.aaroneaton.com/v1alpha1
kind: DealTheTurn
metadata:
  name: tour
  namespace: <shared namespace>
spec:
  tableName: <providerConfig name>
  writeConnectionSecretToRef:
    name: river-card
```

Players can investigate TheRiver by running:
```
kubectl get secret river-card -n <shared namespace> -o yaml
```

### How do I stop someone from cheating?

RBAC Controls and "Pit Bosses."

Your players should only have the ability to create the Claims for the Composites we
are building. Since the actual cards are Cluster-Scoped (like all managed
resources), working directly with cards requires admin privileges. Your Players
should not be admins. They only have access in their own namespace
and the shared "table" namespace.

Meanwhile, cluster administrators can see every card at the table. So if a player
somehow manages to pull a card directly the cluster administrators would see
it happen and intervene.

Let's get back to the demo.

### An Imperfect Endgame

Now it is time for everyone left to _show_ their cards. 

Crossplane Compositions really aren't intended for "sharing" information. The
design originally assumed that everyone who created infrastructure wanted to
_own_ that infrastructure and there was little need for community access.

In practice, teams often create message queues or share data buckets to
facilitate communication. When this happens the owning team wants to share
connection secrets into another team's namespace.

I've opened an issue on [the crossplane
repo](https://github.com/crossplane/crossplane/issues/2929) to provide this
functionality.  This has some significant security implications, of course.

For now, here is my imperfect "workaround" that will allow the game to complete:

Grant The Dealer access to edit the PlayerHands.

When everyone has called, the dealer will edit the hand of everyone still in the
game, changing the name and target of the name and namespace of the
__writeConnectionSecretToRef__ field.

If The Deal needs to know the name of a player, it is shown in the
__crossplane.io/composite__ annotation on any of a player's cards. 

If the annotation on __playerone's__ card is
"crossplane.io/composite=my-hand-blm2v", then:

```yaml
---
apiVersion: cardgames.aaroneaton.com/v1alpha1
kind: PlayerHand
metadata:
  name: my-hand-blm2v
spec:
  playerName: <player-name>
  tableName: <providerConfig name>
  writeConnectionSecretToRef:
    name: playerone-cards # edited to player's name
    namespace: <shared namespace> # edited to shared namespace
```

Now... can we make it play Blackjack?

![Can you make it play
Blackjack?](../../images/crossplane/crossplane-configuration-development-with-poker/composite-blackjack.png
"Can you make it play Blackjack?")

## Useful Links

- [Crossplane Configuration Docs](https://crossplane.io/docs/v1.6/getting-started/create-configuration.html) 
- [Kuttl configuration reference](https://kuttl.dev/docs/testing/reference.html) 
- [Crossplane Terminology](https://crossplane.io/docs/v1.6/concepts/terminology.html#crossplane-terms)
