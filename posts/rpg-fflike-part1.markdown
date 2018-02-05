---
title: RPG with Player-Defined AI - Part 1
date: 2018-01-14
published: false
---

I tried to pick a fun theme for this. I think RPGs are fun, you (hopefully) think RPGs are fun, everyone (maybe) thinks RPGs are fun. So I'll do an RPG but give it a twist so it's not just another entry in the plethora of "how to do an RPG in Unity" tutorials out there. In fact, this won't be that much of a tutorial, more like an implementation overview and discussion.

It's all about Unity and C#. Readers at all levels should be able to extract ideas from this, but for the best value for your time you want to have at least some experience with Unity and an intermediate understanding of C#. It may help to also be familiar with the basic concepts of functional programming.

Before we begin, most of what I'll show here uses visual assets obtained from Unity's store. I just mixed and matched models and visual effects. Special thanks goes to **Cedric Hutchings** for the Robin Hood-style hat he modeled for one of the skeletons. As far as I know, his is at the time of writing the only hat of its kind on the web! You can find it here: [OpenGameArt.org](https://opengameart.org/content/robin-hood-hat)

Let's go!

## What are we making?

A real time party based RPG with player-defined scripts for their characters' behavior in combat. That doesn't sound like a novel concept and indeed these mechanics have been a thing for a long time. It's just that in some games they were more central to the gameplay than in others. What we're going for is something similar to the *gambits* in Final Fantasy XII. Or at least my recollection of them.

<center><a target="_blank" href="/images/ffgambits.jpg"><img style="width:600px" src="/images/ffgambits.jpg" /><a/></center>
<center>*Final Fantasy XII; Source: Mobygames*</center>

The player will control a party that consists of, at least for now, three characters: a Dragon Knight, a Sorceress and a Priest. Excellent synergy, I'd have added a Ranger too but there were no Rangers on the asset store that would try to fit in with the rest of the assets. A pity, but it's not a crucial flaw. There's a skeleton shooting bolts at us later down the line, for fans of ranged weaponry. Besides, I only had one ranger hat and the skeleton claimed it first.

The player can directly control one of the characters at a time, and the rest of the party will follow. When in combat, the player can give orders to the selected character, while the others will follow a player-defined combat routine.

For this first part, the party is set to do some combat with a skeleton band that consists of a skeleton archer (actually using a crossbow, but I doubt many people search for "skeleton crossbowman" on Google), a skeleton warrior and a skeleton. The last guy's equipment budget was even lower than my assets budget, in case you were wondering.

<center><a target="_blank" href="/images/battle.png"><img style="width:600px" src="/images/battle.png" /><a/></center>
<center>*Our game! The battle rages on...*</center>

## Let's describe the whole AI thing

If you've gone this far chances are you want to hear about the player-scriptable combat AI, unless you can't get enough of articles showing you how to do an inventory system, which is fine I don't judge, but that comes a bit later.

I'll first describe how the combat system ought to work and then try to put it in context, to show how it ties to the rest of the game.

I mentioned Final Fantasy XII as a source of inspiration, and if you checked out the screenshot I also added you may have already gotten an idea of where we're going with this.

Each party character (and NPC, though the player doesn't see that) has a sort of *combat script* attached to them, which is really a list of rules they must follow when doing combat. We can call the rules `if-then` rules which is accurate enough especially in the context of FF XII, though we'll see that during implementation we gain a bunch of benefits if we think of them as something more along the lines of `filter-select-maybeThen` rules.

Here's a small script:

```java
(Ally: HP < 30%) -> Cast Heal
(Enemy: Nearest) -> Attack
```

Every time a character finds itself to be sitting idle in combat, they'll execute this script, trying to find a valid action to perform next. They try out `(Ally: HP < 30%)` but there's no ally with less than 30% hitpoints in combat. So they find the nearest enemy to them via `(Enemy: Nearest)`, and jump in for one attack. After that attack finishes, they re-run the script, and pick the *first action whose condition passes*.

This is how we define a `CombatRule`:

```cs
// Self in this context represents the GameCharacter that owns
// the combat AI rule.
// Some filters are only relevant relative to him/her 
// e.g. FarthestCharacter
public class CombatRule
{
    private GameCharacter _self;
    private IList<ICharacterFilter> _filters;
    private ITargetSelector _selector;

    private Func<GameCharacter, GameCharacter, CombatAction> 
        _actionCreator;

    // Implementation here.
}
```

From a programming standpoint, there's really no difference between `ITargetSelector` and `ICharacterFilter`. The `ITargetSelector` acts like a first filter, which decides whether the rule will apply the filters that follow to `Self`, `Enemies` or `Allies`. Having this as the first required filter avoids issues of ambiguity (imagine a rule with `HP >30%` as the only filter and no target selector: allies, self and enemies can make the check pass — which do we choose?) so it's mostly a thing to make programming and testing rules less bug prone in the future.

^ BAD SOUNDING

`ICharacterFilter` asks that implementers feature a 

```cs
IEnumerable<GameCharacter> Filter(GameCharacter self, 
    IEnumerable<GameCharacter> characters);
```

function. Given the shape of the function, we can easily chain filters together. Nearest character that has health below 50 percent? Run the `StatsBelowPercent` filter to get the characters that have health below 50%, then run the `NearestCharacter` filter to pick the one closest to `Self`.

Filter implementation is straightforward. Here's one, that picks the nearest character to `Self`:

```cs
public class NearestCharacter : ICharacterFilter
{
    private float DistanceBetween(GameCharacter c1, GameCharacter c2)
    {
        return Vector3.Distance(c1.transform.position,
            c2.transform.position
        );
    }

    public IEnumerable<GameCharacter> Filter(GameCharacter self, 
        IEnumerable<GameCharacter> characters)
    {
        Func<GameCharacter, float> distance = 
            (other) => DistanceBetween(self, other);

        return new[] { characters.MinBy(distance) };
    }
}
```

<div class="in-article-question">
In case you were wondering, `MinBy` up above is brought by [MoreLINQ](https://github.com/morelinq/MoreLINQ). Together with his brother `MaxBy`, they're two very useful operators that unfortunately aren't part of LINQ proper. Finding the object that has the least or most of something is a pattern that shows up in my projects all the time.
</div>

Filters are not commutative:

```cs
// Fails if the nearest character's health
// isn't below 60%.
NearestCharacter() -> StatsBelowPercent(Health, 60)

// Fails only if there are no characters
// with health below 60%.
StatsBelowPercent(Health, 60) -> NearestCharacter()
```

so the order they're executed in (which depends on the order they're appended to a `CombatRule` in) matters.
