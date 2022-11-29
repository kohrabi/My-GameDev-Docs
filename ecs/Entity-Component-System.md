
### Source:
https://austinmorlan.com/posts/entity_component_system/

# Brief Definition
[[Entity-Component-System#Entity|Entity]] is an ID. Entity is used as a pointer to the right set of [[Entity-Component-System#Component|Component]] and [[Entity-Component-System#Signature|Signature]]
[[Entity-Component-System#Component|Component]] is a struct of data
[[Entity-Component-System#Signature|Signature]] is a bitset of which components the entity has
**System** is the logic that operates on the components
[[Entity-Component-System#Entity Manager|Entity Manager]] is in charge of distributing entity IDs and keeping record of which IDs are in use and which are not.
[[Entity-Component-System#Component Manager|Component Manager]] is a simple array, but is always a _packed_ array, meaning it has no holes
# Entity
```C++
// A simple type alias 
using Entity = std::uint32_t; 
// Used to define the size of arrays later on 
const Entity MAX_ENTITIES = 5000;
```
[[Entity-Component-System#Brief Definition|Entity]] could be in any size, and same with **MAX_ENTITIES**. ^MAXENTITIES
# Component
[[Entity-Component-System#Brief Definition|Component]] is a struct with a small chunk of functionally related data

As an example, **Transform** might look like this:
```C++
struct Transform 
{ 
	Vec3 position; 
	Quat rotation; 
	Vec3 scale; 
}
```
Each [[Entity-Component-System#Brief Definition|Component]] type (**Transform**, **RigidBody**, etc) also has a unique ID given to it
```C++
// A simple type alias 
using ComponentType = std::uint8_t; 
// Used to define the size of arrays later on 
const ComponentType MAX_COMPONENTS = 32;
```
[[Entity-Component-System#Brief Definition|ComponentType]] could be in any size, and same with **MAX_COMPONENTS**.
# Signature
Each [[Entity-Component-System#Brief Definition|Component]] type has a unique ID (starting from 0), which is used to represent a bit in the signature.
As an example, if **Transform** has type 0, **RigidBody** has type 1, and **Gravity** has type 2, an entity that “has” those three  [[Entity-Component-System#Brief Definition|Components]] would have a signature of 0b111 (bits 0, 1, and 2 are set).
```C++
// A simple type alias 
using Signature = std::bitset<MAX_COMPONENTS>;
```
Certain [[Entity-Component-System#Brief Definition|System]] will have a predetermined [[Entity-Component-System#Brief Definition|Signature]] showing which components does the [[Entity-Component-System#Brief Definition|System]] need. For example, Let the bitset of compents be 0b0000 (0 is **Transform**, 1 is **Rigidbody**, 2 is **Gravity**, 4 is **Animated**) **Rigidbody**, **Transform** and **Gravity** are gonna be need for *Rigidbody System* so the bitset gonna be 0b1110, **Animated**, **Transform** is gonna needed for *Animation System*, ec.

A simple bitwise comparision can ensure that an entity's signature contains the [[Entity-Component-System#Brief Definition|System]]'s [[Entity-Component-System#Brief Definition|Signature]].
(an entity might have _more_ components than a system requires, which is fine, as long as it has _all_ of the components a system requires).
# Entity Manager
A simple **std::queue**, where on startup the queue is initialized to contain every valid [[Entity-Component-System#Brief Definition|Entity]] ID up to [[Entity-Component-System#^MAXENTITIES|MAX_ENTITIES]]. 
```C++
// Queue of unused entity IDs 
std::queue<Entity> mAvailableEntities{};
```
When an [[Entity-Component-System#Brief Definition|Entity]] is created it takes an ID from the **FRONT** of the queue, and when an entity is destroyed it puts the destroyed ID at the **BACK** of the queue.
```C++
EntityManager() 
{ 
	// Initialize the queue with all possible entity IDs 
	for (Entity entity = 0; entity < MAX_ENTITIES; ++entity) 
		mAvailableEntities.push(entity); 
}
```
# Component Array
If an [[Entity-Component-System#Brief Definition|Entity]] is just an index into an array of components, then it’s simple to grab the relevant [[Entity-Component-System#Brief Definition|Component]] for an entity. When an [[Entity-Component-System#Brief Definition|Entity]] is destroyed, that index into the array is no longer valid.

The entire point of the ECS is to keep the data _**packed**_ in memory, meaning that you should be able to iterate over all of the indices in the array without needing any sort of “if(valid)” checks. When an [[Entity-Component-System#Brief Definition|Entity]] is destroyed, the [[Entity-Component-System#Brief Definition|Component]] data it “had” still exists in the arrays. If a system were to then try to iterate over the array, it would encounter stale data with no [[Entity-Component-System#Brief Definition|Entity]] attached. For this reason we need to keep the array packed with valid data at all times.

You do this by keeping a mapping from [[Entity-Component-System#Brief Definition|Entity]] IDs to array indices. When accessing the array, you use the [[Entity-Component-System#Brief Definition|Entity]] ID to look up the actual array index. Then, when an entity is destroyed, you take the last valid element in the array and move it into the deleted entity’s spot and update the map so that the entity ID now points to the correct position. There is also a map from the array index to an entity ID so that, when moving the last array element, you know which entity was using that index and can update its map.