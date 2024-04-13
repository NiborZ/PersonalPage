To become proficient with the specific mechanisms of monster/NPC pathfinding allows for more deft customization of NPC pathfinding-related functions. This article aims to deconstruct the C++ layer pathfinding mechanisms, enabling readers to understand which aspects of the mechanism are involved, quickly locate the parts they wish to change, and ultimately achieve a more advanced and distinctive pathfinding system.

In games, on the client-side, we often use the Navigational Mesh to drive the movement of NPCs or monsters, which in turn moves the IEntity in the game world, allowing the IEntity to demonstrate autonomous movement capabilities. Typically, this is used in the following situations:
- The server syncs the movement target points to the client, where client-side monsters/NPCs autonomously navigate to the target point location.
- Pure client-side AI drives monsters/NPCs to navigate to a point.

Messiah's pathfinding solution is Recast/Detour. This article details how Navigator and CharCtrl determine the final position each frame and assign it to IEntity during the movement driven by pathfinding.

**Python Layer Initialization Work**
Assume the Python code has already created a Monster, inherited from ClientAreaEntity. Suppose Monster.model is a Python layer Model class object, and assume Monster.model.model has already been created as a C++ layer IEntity object and has attached Model Component and ActorComponent, and set Skeleton and Graph for the ActorComponent. Then we create a CharCtrlComponent:
```python
def CreateCharCtrl(collision_info=None):
	import MObject
	charctrl = MObject.CreateObject('CharCtrlComponent')
	charctrl.CollisionFilterInfo = 31
	charctrl.MaxSlope = .9
	charctrl.Gravity = MType.Vector3(0, 0, 0)
	charctrl.Up = MType.Vector3(0, 1, 0)
	charctrl.CreateCapsuleCharCtrl(.3, .5)
	return charctrl
```
Assuming `self` is Monster.model, create CharCtrl and associate it with the IEntity type model variable:
```python
        self.charctrl = MHelper.CreateCharCtrl(None)
        self.model.CharCtrl = self.charctrl
```
Then create Navigator and associate it with CharCtrl:
```python
	self.navigator = MObject.CreateObject('INavigatorComponent')
	self.navigator.AngularSpeed = 15
	self.navigator.ObstacleRadius = 1
	self.navigator.ObstacleClass = 4
	self.navigator.ownerid = entity.id
	self.navigator.EnableObstacle = bool(self.navigator.ObstacleRadius)
	# Bind Navigator to CharacterController
	self.charctrl.Navigator = self.navigator
```
At runtime, we can drive Monster's movement through Navigator (assuming `self` is Monster.model):
```python
	self.navigator.MoveTo(MType.Vector3(*position)) # Set the pathfinding target point
	self.navigator.BindEvent("Arrived", self.navigatorEndNotify) # Callback after reaching the target point
	self.model.Skeleton.ApplyMotion = False # Disable motion data driving displacement
```
This Monster can now move along the Nav Mesh. Python layer does not see the specific Monster movement mechanism code, because the actual mechanism is implemented in the C++ layer. Below we first look at the specific steps of C++ layer pathfinding changing Monster.model.model.

**C++ Layer Overall Process**
The first step is that the Python layer calls MoveTo() function which actually calls the C++ layer Recast's MoveTo() function. The call stack is as follows:
```cpp
	RecastExt::extCrowd::requestMovePos(const unsigned short handle, unsigned int ref, const float* pos, const float range)
	RecastExt::NaviMapper::RequestMovePos(const unsigned short handle, unsigned int ref, const RecastExt::Point3 & pos, const float range)
	RecastNavigator::MoveTo(const Messiah::TVec3<float> & dst)
	INavigatorComponent::MoveTo(const Messiah::TVec3<float> & dst)
	PyINavigatorComponent::_ImpMoveTo<void>(const Messiah::TVec3<float> & _WrapParam1)
	PyINavigatorComponent::MoveTo(const Messiah::TVec3<float> & _WrapParam1)
```
Here in the extCrowd::requestMovePos function there is `dtVcopy(ag.targetPos, pos);` which modifies the Agent's pathfinding target position ag.targetPos. The concept of Agent will be explained later.
The second step: Call extCrowd::update in each frame of RecastMap::Tick() to update the pathfinding route and the latest pathfinding position.
```cpp
	RecastExt::integrate(RecastExt::CrowdAgent * ag, const float dt)
	RecastExt::extCrowd::update(const float dt, RecastExt::CrowdAgentDebugInfo * debug)
	RecastExt::NaviMapper::Update(const float dtime)
	Messiah::RecastMap::Tick(float dtime)
	Messiah::INavigateMap::_PostTick_on_ot(floatdtime)
```
In the integrate() function, `dtVmad(ag->npos, ag->npos, ag->vel, dt);` updates the new pathfinding position of the Agent.

The third step: Pass the Agent's npos to CharCtrlComponent.mMoveTargetPos, specifying the location CharCtrl is about to move to:
```cpp
CharCtrlComponent::SetMoveTarget(const Messiah::TVec3<float> & pos, float yaw)
NavigateCharCtrl::MoveTo(const Messiah::TVec3<float> & pos, float yaw)
RecastNavigator::Tick(float dtime)
INavigatorComponent::_PostTick_on_ot(float dtime)
```

The fourth step: In CharCtrlComponent::RealTickInSimDropTestMode_on_ot(), perform downward collision detection based on mMoveTargetPos to stick to the ground, then set the post-collision coordinates for CharCtrl:
```cpp
Messiah::CharCtrlComponent::RealTickInSimDropTestMode_on_ot(float dtime)
Messiah::CharCtrlComponent::RealTick_on_ot(float dtime, bool & needTriggerFalling)
Messiah::CharCtrlComponent::_UpdateCharCtrl_on_ot(float dtime)
Messiah::NavigateCharCtrl::Tick(const float dtime)
Messiah::RecastNavigator::Tick(float dtime)
Messiah::INavigatorComponent::_PostTick_on_ot(float dtime)
```

The fifth step: Set the CharCtrl's position on the corresponding IEntity.
```cpp
CharCtrlComponent::UpdateEntityAfterRealTick_on_ot(Messiah::IEntity* entity)
CharCtrlComponent::_UpdateCharCtrl_on_ot(float dtime)
```
Here, the actual execution of setting the IEntity's position is `entity->SetPropertyValue<Matrix4x3>(EName_Transform, transMat);`

Thus, the position of the IEntity is set.

Next, we will focus on the extCrowd::update() function, which is the core of the pathfinding process, the essence of it. Before we delve into this function, let's first look at some of the basic concepts of Detour to better understand the code.

**Some Basic Concepts in Detour**
CrowdAgent: Every monster/NPC with an INavigatorComponent corresponds to a CrowdAgent. CrowdAgent has a full set of functions and member variables related to pathfinding, completing the pathfinding logic. Hereafter referred to as Agent.

extCrowd: A group of monsters/NPCs form a Crowd. A Crowd can be considered as a group of monsters/NPCs with close spatial positions, which may collide and avoid each other during the pathfinding process because they pass through the same route.

Corridor: A series of adjacent polygons on the pathfinding route of an Agent, strung together to form a "corridor," which is this corridor.

Corner: When the destination point cannot be seen directly from the current position, the pathfinding route will turn. The corner point near the inner circle at the turn is the shortest path that must be passed, which is a corner.

OffMeshLink: To allow monsters/NPCs to navigate between disconnected nav meshes, connection points can be set in the RecastMapEditor between two nav meshes. This connection is an OffMeshLink.

Neighbour: In the same extCrowd, multiple Agents that are close to this Agent and may collide during pathfinding are my Neighbours.

dtNavMeshQuery: A class providing a variety of functions using the Nav Mesh, the most typical of which are findPath() and raycast().

Multiple NavMesh Layers: In RecastMapEditor, you can specify three different AgentRadius to generate Nav Mesh, becoming different Nav Mesh Layers. For example, an ordinary humanoid NPC can use a small Radius to generate a Nav Mesh Layer, while a huge Boss can use a large Radius to generate another Nav Mesh Layer. Different Layers can be accessed in extCrowd::m_navquerys. The index required for access can be obtained from extCrowd::getRadiusLayer(ag.params.radius) for a specific radius corresponding to the nav mesh layer of an agent.

Y-Axis Ignored: In detour, the Y-axis is mostly not included in calculations, mainly calculations are done on the xz plane.

Getting the Corner Points on the Pathfinding Route: RecastNavigator::GetWayPoints() can provide the corner points on the pathfinding route. It can be used for route processing (such as sending from server to client to move NPC) and debug display.

**C++ Layer Process of Agent Movement on Nav Mesh Each Frame**
The main update function for crowdAgent to move along the nav mesh each frame is extCrowd::update(), below we detail each step of this function.
```cpp
void extCrowd::update(const float dt, CrowdAgentDebugInfo* debug)
{
	m_velocitySampleCount = 0;
	// Check that all agents still have valid paths.
	checkPathValidity(dt);
	// Update async move request and path finder.
	updateMoveRequest(dt);
	// Check that all agents still have valid paths.checkPathValidity(dt);
    // Update async move request and path finder.updateMoveRequest(dt);
    // Optimize path topology.updateTopologyOptimization(dt);
    // Update the neigbour managerm_neigbourManager->Update();
    // Get nearby navmesh segments and agents to collide with.updateNeighbours();
    // Find next corner to steer to.findSteerCorner(debug);
    // Trigger off-mesh connections (depends on corners).triggerOffmeshConn();
    // Calculate steering.calcSteering(dt);
    // Velocity planning.	planVelocity(dt, debug);
    // Integrate.// 此处省去若干行integrate(&ag, dt);
    // Handle collisions.recoverCollision();
    // Move along navmeshmoveAlongMesh(dt);
    // Check whether arrived.checkArrived();
}
```
recast/detour provides the functionality to dynamically set obstacles and dynamically regenerate the nav mesh inside local tiles. Therefore, there's a possibility that the corridor of the Agent from the previous frame was valid, but the polys on the corridor of this frame have disappeared, making it invalid. If it becomes invalid, we need to notify subsequent steps to replan the entire corridor:
• Check if the first poly of the corridor is valid. If it's invalid, find a new nearest poly based on the current ag.npos (Agent's current position) using dtNavMeshQuery::findNearestPoly(), set it as the first poly of the corridor (dtPathCorridor::fixPathStart()), and set replan=true.
• If there are other invalid polys on the corridor, except the first one (dtPathCorridor::isValid()), set replan=true.
• If the last poly of the corridor differs from the newly found ag.targetRef, set replan=true.
• If replan=true, then call extCrowd::requestMoveTargetReplan() to initialize the state before recalculating the route:
   - ag.targetPathqRef = DT_PATHQ_INVALID
   - ag.targetReplan = true
   - ag.targetState = ECROWDAGENT_TARGET_REQUESTING

extCrowd::updateMoveRequest() handles path requests, serving as the main entry point for route calculation:
• If ag.targetState is ECROWDAGENT_TARGET_REQUESTING, recalculate the corridor:
   - dtNavMeshQuery::initSlicedFindPath()
   - dtNavMeshQuery::updateSlicedFindPath()
   - dtNavMeshQuery::finalizeSlicedFindPathPartial()
   - ag.corridor.setCorridor() sets the new corridor here
• If there are unresolved path calculation requests (extCrowd::m_numPathqs), calculate those routes: dtPathQueue::update()
• Process the calculated routes, finally setting ag.corridor.setCorridor(targetPos, res, nres) to complete setting up the corridor and setting ag.targetState = ECROWDAGENT_TARGET_VALID

dtPathCorridor::optimizePathTopology() optimizes the path:
After dynamic obstacle avoidance, the Agent's position may no longer be the same as before. At this point, there's a possibility to optimize the route, for example, if the original route couldn't see the second corridor corner but after dynamic obstacle avoidance, the Agent's position changed, and the corner becomes visible, it's necessary to optimize the route to skip the first corner and head directly to the second one. The optimization can be performed every OPT_TIME_THR seconds (currently set to 5 seconds) or by adding OPT_MAX_AGENTS to the optimization waiting queue every OPT_TIME_THR seconds using RecastExt::addToOptQueue(). This consideration is made for scenarios involving a large number of Agents, such as tens of thousands of them.
The optimization is performed for Agents in the nqueue, with the critical function being ag->corridor.optimizePathTopology().

BoxPrunerNeighbourManager::Update() identifies neighboring agents that might collide with the Agent:
During Agent movement, it's highly likely to encounter position overlaps with other Agents in the vicinity, resulting in collisions. Therefore, it's necessary to determine which neighboring Agents might cause overlaps for handling in subsequent steps.
To control performance consumption, the default setting now is to search for a maximum of 6 potential colliding neighbors in the vicinity: RecastExt::INeighborManager::ExpectedNeighbourCnt = 6

extCrowd::updateNeighbours() locates nearby navmesh segments and Agents for future collision checking:
If the distance between the center of Agent's LocalBoundary and the current ag.npos is greater than ag.params.collisionQueryRange*0.25, we consider recalculating the LocalBoundary of this Agent. A dtLocalBoundary includes the following information:
• m_center: The center point position of the LocalBoundary
• m_segs: The list of edges of polys around the center point, with a maximum of MAX_LOCAL_SEGS, any exceeding are discarded.
• m_polys: The list of polys around the center point, with a maximum of MAX_LOCAL_POLYS, any exceeding are discarded.
The critical function for recalculating the LocalBoundary is dtLocalBoundary::update().
If ag.params.enableObstacle is true, check if there are neighboring Agents nearby using the function QueryNeighbours. In BoxPrunerNeighbourManager::QueryNeighbours(), it directly returns true without performing any action; this is generally the chosen manager. Another option is GridNeighbourManager::QueryNeighbours(), which identifies neighboring Agents that may overlap with the Agent and returns them. Which NeighbourManager to use is specified in the code:
 
static bool UseGrid = false;
// CreateNeigbourManagerINeighbourManager* RecastExt::CreateNeigbourManager(extCrowd& crowd, intagentMaxNeighbours)
{
    if(UseGrid)
        return new GridNeighbourManager(crowd);
    else
        return new BoxPrunerNeighbourManager(crowd, agentMaxNeighbours);
}
Since UseGrid defaults to false, the default choice is BoxPrunerNeighbourManager.

extCrowd::findSteerCorner() determines which corner the Agent should steer towards:
It's necessary to identify the corner towards which the Agent should proceed. Although the corridor has already been identified earlier, it's possible that the corner the Agent is currently heading towards has been blocked by another Agent, so it's necessary to reevaluate which corner to head towards:
• First, use ag.corridor.findCorners() to find the first AGENT_MAX_CORNERS=4 corners.
• Check if there's a neighboring agent standing very close to the first corner. If so, set ag.moveTargetState to ECROWDAGENT_TARGET_NONE or EMOVE_WAITING. This will require recalculating the route in the next frame.
• If the first corner isn't blocked by a neighboring agent, check if the second corner is directly visible. If it is, update the corridor. The critical function here is ag.corridor.optimizePathVisibility().
If neither a neighboring agent is occupying the first corner nor the second corner is directly visible, maintain the current corridor, and the Agent continues towards the first corner.

extCrowd::triggerOffmeshConn() determines if the Agent should enter the movement state of crossing an offMeshConnection.

• First, use RecastExt::overOffmeshConnection() to check if the ag->cornerFlags[ag->ncorners - 1] of the Agent is a DT_STRAIGHTPATH_OFFMESH_CONNECTION. If it is, and if this offsetConnection's corner is very close to the Agent's ag->npos, then it's considered necessary to traverse this offmeshConnection. Here, we can observe that if a corridor passes through an offmeshConnection, the last corner of its corridor is this offmeshConnection.
• If indeed the Agent needs to pass through this offmeshConnection, call ag.corridor.moveOverOffmeshConnection() to obtain where agAnim should start and end (here implying that an animation needs to be played through the offmeshConnection).
• Set the following values related to crossing the OffMeshConnection to ensure correct displacement later:
      - dtVcopy(anim.initPos, ag.npos);
      - anim.polyRef = refs[1];
      - anim.active = true;
      - anim.t = 0.0f;
      - anim.tmax = (ag.params.maxSpeed == 0.0f) ? 0.0f : (dtVdist2D(anim.startPos, anim.endPos) / ag.params.maxSpeed) * 0.5f;
      - ag.state = ECROWDAGENT_STATE_OFFMESH;
      - ag.ncorners = 0; // After passing the offmeshconnection, corners need to be found again
      - ag.neis.clear(); // After passing the connection, neighbors need to be found again

extCrowd::calcSteering() calculates the Agent's new velocity vector based on ag.targetPos (the position of the next corner on the corridor):
• If a smooth turn is required, instead of directly turning, calculate a new normalized direction dvel based on the current dt. Here, the default value of dt is set to the initial value of 0.01666 seconds for 60 frames per second. The key function is RecastExt::calcSmoothSteerDirection(). What's perplexing is that dt isn't actually used in this function, which requires further investigation to understand.
• If a smooth turn isn't needed, calculate the normalized direction dvel directly towards ag.targetPos.
• Check if the final destination is within ag.params.maxSpeed*dt distance, i.e., within the range of one frame movement distance. If it is, set speedScale as getDistanceToGoal()/slowDownRadius and set the length of dvel as ag.params.maxSpeed*speedScale. This ensures that the next frame's movement distance using dvel lands exactly at the final destination.
• Assign the resulting dvel from the above three points to ag.dvel, completing the calculation of the new velocity vector's magnitude and direction.

extCrowd::planVelocity() dynamically avoids obstacles based on the current ag.dvel to calculate ag.nvel (the new velocity vector):
If there are other Agents in my vicinity (neighbors), consider their positions and sizes as individual circles, then use detour's dynamic obstacle avoidance algorithm in dtObstacleAvoidanceQuery::sampleVelocityAdaptive() to find a suitable position around the neighbors to move towards. The resulting new velocity magnitude and direction are stored in ag.nvel. If there are no neighbors nearby, set ag.nvel = ag.dvel directly.

RecastExt::integrate() computes the Agent's final new position:
• Check if the Agent's orientation has changed sides: for example, if it initially veered right but needs to veer left on the next frame, this change is saved in ag.sideChange.
• Set ag.vel = ag.nvel, ultimately setting the Agent's new velocity.
• Set ag.npos = ag.npos + ag.vel*dt, ultimately setting the Agent's new position. This is crucial because CharCtrl will use this ag.npos to move the CharCtrl in the future.
• Set ag.yaw to the direction pointed by ag.vel, ultimately setting the Agent's new yaw value.

extCrowd::recoverCollision() prevents Agents from overlapping:
After dynamic avoidance, if Agents have moved to new positions, there might be an issue of Agents overlapping each other (the current dynamic avoidance algorithm isn't perfect, achieving good dynamic avoidance requires higher-level coordination above Agents). In this function, treat the neighbors as 2D circles. If the distance between me and them is less than the sum of their radii, they are considered overlapping. Then, calculate a displacement vector ag.disp and add this ag.disp to ag.npos to form the new Agent position.

extCrowd::checkArrived() checks if the destination has been reached, then updates the state:
If the distance between ag.npos and ag.targetPos is less than a certain value, it's considered that the destination has been reached.
        ag.targetState = ECROWDAGENT_TARGET_NONE;
        ag.moveState = EMOVE_NONE;
Updates to these two states are crucial since detour's various stages are determined based on targetState and moveState.

Display Debug Information:
To facilitate debugging the Nav Mesh, we can display it for observation. Below is an example of a GM command. Input "!togglenavmeshdisplay" in the Console window under the Messiah SDK. Note that "!" is used here instead of "$", as the latter would be interpreted as a server command, even though "$ToggleNavMeshDisplay" is used in GM command code.