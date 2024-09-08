It is a popular approach to manage game data. It efficiently handles large amounts of data and is becoming the de facto standard for large engines such as [[Unity]] and [[Godot]] ([[Unreal Engine]] uses similar system with components, but without separate systems). [[Rust]] is a great fit for ECS-driven game development, and you can find several great ECS systems available in the crates system.

# Terminology

## Entity
An *entity* can be anything: and advanturer, an orc, or a pair of shoes. The game map is an exception - it's usually not an entity, but rather a source entity's reference to travel. Entities don't have logic associated with them; they are little more than an identification number.
## Component
A *component* describes a property an entity may have. entities typically have lots of components attached to them, serving as a description - and adding functionality through systems. For example, a goblin might have a *Position* component describing where it is on the map, a *Render* component describing how to render it, a *MeleeAI* component indicating that it attacks with hand-to-hand combat, and a *Health* component describing how many hit points it has left. Components don't have logic associated with them, either.
## Systems
*Systems* query the entities and components and provide one element of game-play/world simulation. For example, a *Render* system might draw everything with a *Render* component and a *Position* component onto the map. A *Melee* system might handle hand-to-hand combat. Systems provide the "game logic" that makes your game function. Systems read entity/component data and make changes to them - providing the powerhouse that makes the game function.
## Resources
*Resources* are shared data available to multiple systems

![[ECS_terms_relationships.png]]
# Composing Entities
- Entities are composed of combined components that describe them. 
- Whenever you add a component type, you're offering that functionality to any entity
- One advantage of using component composition is that it becomes easier to translate from an English description to a set of components

**Example:**
"Goblin is a small, green, angry humanoid. They roam the dungeon, preying upon unsuspecting adventurers. They prefer to attack from a distance, and are cowardly and quite weak."

- It's small humanoid requiring the same *Render* components as other humanoids
- It has a *Position* on the map
- It's angry, so you need an AI component denoting that it attacks on sight
- It prefers ranged attacks, so you AI components should indicate that
- It is cowardly, so maybe that needs a component when you implement running away
- It is quite weak, implying that while it has a *Health* component, the number of hit points is quite low.
![[component_composition.png]]