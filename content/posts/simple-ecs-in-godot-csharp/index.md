---
date: '2026-04-08'
draft: false
title: 'Simple ECS in Godot C#'
tags: ["Godot", "C#"]
---

## Why Entity Component System ?

As we known, Entity Component System (ECS) is widely used in game dev, like Unity's C# script system, or Unreal's Actor-Component structure.
ECS is well-known for its flexibility, building a single entity with several pre-made components.

For a simple example, you wanna make a cat entity. Just pickup these common components: model_comp (handle model and material display), animation_comp (handle cat's animation), sound_comp (handle meow~), ai_comp (decide where to walk to and what cat's doing). And finally, you set up cat's data into these components, like cat's model resource path, cat's sound path, the ai behave tree path and animation state machine path.

In this example, all of the components is pre-made and well-designed. So it is easy to make a cat entity. Just put components together and inject some cat's proprietary config data.

Based on ECS, you can make a specific entity at run-time, even this entity is not pre-defined in source code. I mean, if you input the name of needed components and input config data (like a recipe), then the system will just make it for you. This can be called as a class factory design.

The re-usable components and run-time flexibility are solid reasons why I love ECS.


## ECS for Godot C#

Godot has node system, and one single node can only be attached by one single script. So it is easy to find out that one node stands for one component. This is my player scene sctructure:

![](pic0.png#width=400)

As a FPS player, it need movement, left & right weapons, and HUD components, each component handle its own logic independently for most of the time. When they need to connect with each other, the componnet first call owner entity, and find the other component by the type name.

Now take a look at IEntity and IComponent interfaces.

```csharp
public interface IEntity
{
	Dictionary<Type, IComponent> Components { get; }
}

public interface IComponent
{
	IEntity Entity { get; set; }

	void OnComponentAdded();
	void OnComponentRemoved();
}
```

And here is the component add & get functions.

```csharp
public static class EntityExtension
{
	public static void AddComponent(this IEntity entity, IComponent component)
	{
		Type type = component.GetType();
		if (entity.HasComponent(type))
		{
			GD.PrintErr("AddComponent: duplicated components <" + type.Name + ">, remove old one.");
			entity.RemoveComponent(type);
		}
		entity.Components[type] = component;
		component.Entity = entity;
		component.OnComponentAdded();
	}

	public static void AddComponent<T>(this IEntity entity, T component) where T : class, IComponent
	{
		entity.AddComponent((IComponent)component);
	}

	public static T GetComponent<T>(this IEntity entity) where T : class, IComponent
	{
		if (entity.Components.TryGetValue(typeof(T), out var component))
		{
			return (T)component;
		}
		return null;
	}

	public static void RemoveComponent(this IEntity entity, Type type) 
	{
		if (entity.Components.TryGetValue(type, out var component))
		{
			component.OnComponentRemoved();
			component.Entity = null;
			entity.Components.Remove(type);
		}
	}

	public static void RemoveComponent<T>(this IEntity entity) where T : class, IComponent
	{
		entity.RemoveComponent(typeof(T));
	}

	public static bool HasComponent(this IEntity entity, Type type)
	{
		return entity.Components.ContainsKey(type);
	}

	public static bool HasComponent<T>(this IEntity entity) where T : class, IComponent
	{
		return entity.HasComponent(typeof(T));
	}

	public static void InjectComponents(this IEntity entity)
	{
		if (entity is not Node entityNode)
		{
			GD.PrintErr($"InjectComponents: {entity.GetType().Name} is not godot node type");
			return;
		}

		Node componentsNode = entityNode.GetNodeOrNull<Node>("Components");
		if (componentsNode is null)
		{
			GD.PrintErr($"InjectComponents: {entity} has no 'Components' node.");
			return;
		}

		foreach (var child in componentsNode.GetChildren())
		{
			if (child is IComponent component)
			{
				entity.AddComponent(component);
			}
			else
			{
				GD.PrintErr($"InjectComponents: {entity} has non-component node '{child.Name}' inside 'Components' node.");
			}
		}
	}
}
```

Then if you wanna make a new component, just write like this. The`Entity` attribute will be set at entity's init stage.

```csharp
public partial class HUDComponent : Node, IComponent
{
	// interface
	public IEntity Entity { get; set; }

	// interface
	public void OnComponentAdded()
	{
	}

	public void OnComponentRemoved()
	{
	}

	public override void _Ready()
	{
	}

	public override void _Process(double delta)
	{
	}
}
```

If you wanna make a new entity, just write like this. `Components` contains all of the components this entity has, and is easy to debug.

```csharp
public partial class PlayerEntity : CharacterBody3D, IEntity
{	
	// interface
	public Dictionary<Type, IComponent> Components { get; } = new();
	
	public override void _Ready()
	{
		this.InjectComponents();
	}

	public override void _Process(double delta)
	{
	}
}
```

At last, for method-calling between two components, write like this.

```csharp
// code inside component
var shootComponent = Entity.GetComponent<ShootComponent>();
shootComponent.ActionAttack();
```