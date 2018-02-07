---
title: RPG Combat AI with If-Then Rules in Unity
date: 2018-02-05
published: false
---

Final Fantasy XII has one of the most interesting features I've seen in games up to date: the Gambit system. It provides a neat way to program the AI of your party's characters using "if-then" rules. It's clever, simple, and perfectly usable. The system is extensively described on the [Final Fantasy wiki](http://finalfantasy.wikia.com/wiki/Gambits), if you're curious about its intricacies.

A while ago I was wondering what it'd take to get something like it up and running in Unity, and the following prototype is the result. It's definitely rough around the edges, but worked on some more it could prove to be a great way to get clever-looking AIs up and running. I can see it being applicable not only to RPGs but also other game genres where there are direct confrontations between multiple characters, with both offensive and support mechanics.

A short gameplay video is here:

<center>
<iframe width="590" height="430" src="https://www.youtube.com/embed/-XMi4J_4RCI" frameborder="0" allowfullscreen></iframe>
</center>

This post will discuss the core of the AI system only — the rest of the systems, even though visible in the gameplay video, will be treated as black boxes. For starters, they're there just to provide something to get the action running. As such, they're incomplete and not very interesting to talk about. They're, however, very likely to be the focus of a few different articles in the future; I feel the prototype is worth working on further.

<div class="in-article-question">
<h3>Note</h3>
Most of what I'll show here uses visual assets obtained from Unity's store. I just mixed and matched models and visual effects. Special thanks goes to **Cedric Hutchings** for the Robin Hood-style hat he modeled for one of the skeletons. As far as I know, his is at the time of writing the only hat of its kind on the web! You can find it here: [OpenGameArt.org](https://opengameart.org/content/robin-hood-hat)
</div>

## Overview

<center>
<img src="/images/skeles.png"/>

*The relevant components of a character doing battle.*
</center>

These are the rules for the prototype:

* Combat is done between `GameCharacter`s of opposing factions (in video you can see the Human and Undead factions at work - they're enemies).
* Each `GameCharacter` has a `CombatAI` component that allows it to perform `CharacterAction`s, meaning actions by characters targeted at characters.
* `CharacterAction`s are performed via the appropriate `CharacterAbility`. `UseItemAction` is thus performed by the `ItemUseAbility`.
* When at any point in time a character finds itself having no assigned `CharacterAction`, the `CombatAI` goes through the character's combat rules and tries to pick one.
* When an action has been picked:
    * The character starts walking towards the target in order to get into the appropriate range for the action.
    * At the same time a delay countdown is started; some actions have a longer delay than others.
    * When the target is in range and the delay has passed:
        * Perform the action
        * When the action has finished performing, go back to picking another.

## The `CombatAI`

The `CombatAI` class encapsulates the logic described in the rules above. It's fairly straightforward:

```cs
// The Combat AI class with non-critical
// code removed.
public class CombatAI : MonoBehaviour
{
    private GameCharacter _self;
    public Gambit CombatGambit;
    private ActionWrapper _activeAction;

    private void PickNextAction()
    {
        var matchedRule = CombatGambit.FindMatchingRule(_self, 
            otherCombatants);
        if (matchedRule == null)
        {
            return;
        }

        var action = matchedRule.Action;
        var target = matchedRule.Target;
        
        // MakeNew returns an ActionWrapper.
        // Explained later on.
        _activeAction = action.MakeNew(_self, target);
        _activeAction.Prepare();

        StartDelay(_activeAction);
    }

    public void Update()
    {
        if (_delayActive)
        {
            UpdateDelay();
        }
        
        if (!delay_Active && _activeAction != null)
        {
            if (_activeAction.NotPerformed() && _activeAction.Ready())
            {
                _activeAction.Perform();
            }
            else if (_activeAction.DonePerforming())
            {
                _activeAction = null;
            }
        }

        if (_activeAction == null)
        {
            PickNextAction();
        }
    }
}
```

Essentially, it takes the `Gambit` assigned to the owning character, and as long as there's no action already chosen, it keeps on trying to get one by executing the rules in the `Gambit`.

## The `Gambit`

This is actually a misnomer, but I decided to roll with it anyway. What FF XII calls *gambits* I call *rules*, and I chose the name *gambit* for the entire set of rules. It doesn't make much sense when you think of it, but the name stuck and causes no issues so with that out of the way...

```cs
public class Gambit : ScriptableObject
{
    public List<CombatRule> Rules = new List<CombatRule>();

    // Self denotes the character who owns the Gambit.
    public RuleMatch FindMatchingRule(GameCharacter self, 
        List<GameCharacter> others)
    {
        foreach (var rule in Rules)
        {
            var target = rule.TryPickingTarget(self, others);
            // CanPerform() represents an implied condition of sorts.
            // Each action type can define its own.
            // But to give an idea, an implied condition for the
            // UseItem action is that the character (self) needs
            // to have at least one of the used item in its inventory.
            if (target != null && rule.Action.CanPerform(self, target))
            {
                return new RuleMatch(rule.Action, target);
            }
        }
        return null;
    }
}
```

We want for the gambits to be tweakable in the editor while the game is running, and also have the changes saved once play mode is exited. Each gambit gets to sit in its own asset, and the same gambit can be assigned to any `GameCharacter`. Hence the `ScriptableObject` approach.

For example, a group of orcs could use only a couple of gambits: `Aggressive Orc Gambit` and `Orc Priest Gambit`. In the prototype scene there's a gambit for each one of the party members (a dragon knight, a sorceress and a priest), and another gambit shared by all the skeletons (which has a single rule: attack the closest enemy), except for the skeleton mage, who has his own gambit.

<center>
<img src="/images/gambit2.png"/>

*A few gambits*
</center>

<center>
<img src="/images/gambit3.png"/>

*A Combat AI component with the Dragon Knight gambit assigned. The `CombatAI` component resides on the `GameCharacter`'s `GameObject`.*
</center>

The gambit has a custom inspector written for it, allowing for easier changing of the rules within.

<center>
<img src="/images/gambit1.png"/>

*The custom inspector for the priest's gambit. It's not very pretty but it does the job.*
</center>

## The `CombatRule`

```cs
public class CombatRule
{
    public GambitTargetSelector TargetSelector;
    public List<GambitTargetFilter> Filters = new List<GambitTargetFilter>();
    public CharacterAction Action;

    // The distinction between Self and Others here is needed:
    // Both the Self selector and some filters are only
    // relevant relative to the given character.
    // For example: Nearest.
    public GameCharacter TryPickingTarget(GameCharacter self, 
        List<GameCharacter> others)
    {
        var candidates = TargetSelector.SelectTargets(self, others);
        if (candidates == null || !candidates.Any())
        {
            return null;
        }
        
        foreach (var filter in Filters)
        {
            candidates = filter.Filter(self, candidates);
        }

        return candidates.FirstOrDefault();
    }
}
```

The `CombatRule` is as plain as it can get. It holds the selector, filters and action and provides an easy way to attempt to find a target among the passed-in characters.

## The `Selector`s and `Filter`s

<div class="in-article-question">
<h3>Note</h3>
Using **LINQ** for the filters was an easy choice, the functional style is the perfect fit. 

In a real project however, this would have to be optimized away; the allocation frenzy just won't scale nicely with the number of characters fighting and with the size of the gambits.
</div>

Selectors and filters are very much alike.

```cs
public abstract class TargetSelector : ScriptableObject
{
    public abstract IEnumerable<GameCharacter> SelectTargets(GameCharacter self, 
        IEnumerable<GameCharacter> candidates);
}

public abstract class TargetFilter : ScriptableObject
{
    public abstract IEnumerable<GameCharacter> Filter(GameCharacter self, 
        IEnumerable<GameCharacter> candidates);
}
```

Selectors pick between `Self`, `Allies` and `Enemies` as the subset of characters to apply filters on. Filters further reduce the space used to pick a target. A `Rule` with no `Filter`s will return the first element of the result of applying the selector onto the list of combatants. 

<center>
<img src="/images/select.png"/>

*Some selector assets.*
</center>

For example, a rule with the `Enemies` selector and no filters will return the first enemy in the list.

A rule with the `Enemies` selector and the `Health below 20%` and `Nearest` filters will return the enemy that has health below 20% and is nearest to the character.

A rule with the `Enemies` selector and the `Nearest` and `Health below 20%` filters will not find a target unless the nearest enemy to the character has health below 20%.

```cs
// The selector for allies.
public class AllyTargetSelector : TargetSelector
{
    public override IEnumerable<GameCharacter> SelectTargets(GameCharacter self, 
        IEnumerable<GameCharacter> candidates)
    {
        return candidates.Where(c => Diplomacy.AreAllies(self, c));
    }
}

// The two filters mentioned above.
public class HealthBelowPercentageGambitFilter : TargetFilter
{
    [Range(0f, 100f)]
    public float Percentage;

    public override IEnumerable<GameCharacter> Filter(GameCharacter self, 
        IEnumerable<GameCharacter> candidates)
    {
        return candidates.Where(x =>
        {
            var stat = x.Stats.Health;
            return stat.Value < (Percentage / 100f * stat.MaxValue);
        });
    }
}

public class NearestFilter : TargetFilter
{
    private float DistanceBetween(GameCharacter c1, GameCharacter c2)
    {
        return Vector3.Distance(c1.transform.position, c2.transform.position);
    }

    public override IEnumerable<GameCharacter> Filter(GameCharacter self, 
        IEnumerable<GameCharacter> others)
    {
        Func<GameCharacter, float> distance = (other) => DistanceBetween(self, other);
        if (others.Any())
        {
            return new[] { others.MinBy(distance) };
        }
        return others;
    }
}
```

<center>
<img src="/images/filters.png"/>

*Some filter assets.*
</center>

The filters and selectors represent most of the boilerplate work of this whole prototype. This is because a new asset/SO instance needs to be created for each separate filter. On the upside, they need to only be created once and you're done.

For some projects a simple health/mana filter split such as 10%, 20%, 30%... may not be enough. In those cases it's very much doable to get to have a single `HealthFilter` or even a general `StatFilter` SO class and have each gambit specify its own values. The basic idea is to decouple the state of the filter from the type of the filter, and treat the filter type as a sort of dictionary key. Essentially, the rules would stop containing filters and instead would contain pairs of filters with their associated data (serialized as part of the gambit that owns it). The custom editor would have to keep these in sync.


## The `CharacterAction`

```cs
public abstract class CharacterAction : ScriptableObject
{
    public abstract string Name { get; }
    public abstract float CombatDelay(GameCharacter self);
    public abstract bool CanPerform(GameCharacter self, 
        GameCharacter target);
    public abstract ActionWrapper MakeNew(GameCharacter self, 
        GameCharacter target);
}
```

There are three types of actions: `AttackAction`, `CastSpellAction` and `UseItemAction`. Each one derives from `CharacterAction` and aside from the `AttackAction`, there's one asset per spell/item.

<center>
<img src="/images/acts.png"/>

*Spell action assets.*
</center>

```cs
public class CastSpellAction : CharacterAction
{
    public override string Name => "Spell: " + Spell.Name;

    public Spell Spell;

    public override float CombatDelay(GameCharacter self)
    {
        return self.Spellcasting.CombatDelayFor(Spell);
    }

    public override bool CanPerform(GameCharacter self, 
        GameCharacter target)
    {
        return self.Spellcasting.CanCast(Spell);
    }

    public override ActionWrapper MakeNew(GameCharacter self, 
        GameCharacter target)
    {
        return new CastSpellActionWrapper(self, target, Spell, Name);
    }
}
```

In a real project the Actions would represent most of the SO asset boilerplate. However, they can all be automatically generated. When adding a new spell or item to the game, automatically generate the `CastSpellAction` or `UseItemAction` assets for it.

## The `ActionWrapper`

The action SOs hold no state related to the characters performing actual actions, they just contain the details for how the action should proceed and a helper for the implied condition checks. 

We need to have a handy way of actually triggering the actions, seeing when they complete and so on. The `ActionWrapper` is essentially a proxy for the `CombatAI` to check on the state of the `CharacterAbility` performing the action. It's meant to only be valid for a single action execution. It's a plain class.

```cs
// Non-critical code removed.
public abstract class ActionWrapper
{
    protected CharacterAbility _ability;

    // ctor omitted

    public void Prepare();
    public bool ReadyToPerform();
    public bool FinishedPerforming();

    public abstract void Perform();
}
```

```cs

public class CastSpellActionWrapper : ActionWrapper
{
    private Spell _spell;

    // ctor omitted

    public override void Perform()
    {
        ((SpellcastAbility)_ability).CastSpellOn(Target, _spell);
    }
}
```