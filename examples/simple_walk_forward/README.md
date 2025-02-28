This is a simple example of using ROS2 to make the robot walk 1m forward.

## Running the Example
1.  Position the robot with 2m of clear space in front of it either sitting or standing (it's going to walk 1m forward)
2.  Make sure you've built and sourced your workspace:
    ```bash
    cd <ros2 workspace>
    colcon build --symlink-install
    source /opt/ros/humble/setup.bash
    source ./install/local_setup.bash
    ```

3.  Define the environment variables `BOSDYN_CLIENT_USERNAME`, `BOSDYN_CLIENT_PASSWORD`, and `SPOT_IP` appropriately for your robot.

4.  Start the driver:
```bash
ros2 launch spot_driver spot_driver.launch.py
```

5.  Run the example:
```bash
ros2 run simple_walk_forward walk_forward
```

The robot should walk forward.

## Understanding the Code

Now let's go through [the code](simple_walk_forward/walk_forward.py) and see what's happening.

Because ROS generally requires persistent things like publishers and subscribers, it’s often useful to have a class around everything.  A [ROS Node](https://docs.ros.org/en/humble/Tutorials/Beginner-CLI-Tools/Understanding-ROS2-Nodes/Understanding-ROS2-Nodes.html) is an object that can interact with [ROS topics](https://docs.ros.org/en/humble/Tutorials/Beginner-CLI-Tools/Understanding-ROS2-Topics/Understanding-ROS2-Topics.html) so our classes usually inherit from Node or contain a node:
```python
class WalkForward(Node):
    def __init__(self, options):
        super().__init__('walk_forward')
```

We first set up ROS's [TF](https://docs.ros.org/en/humble/Tutorials/Intermediate/Tf2/Tf2-Main.html), which helps us transform between robot frames:
```python
         self._tf_listener = TFListenerWrapper('walk_forward_tf', wait_for_transform = [BODY_FRAME_NAME,
                                                                                        VISION_FRAME_NAME])
```
We use a [wrapper](https://github.com/bdaiinstitute/ros_utilities/blob/main/bdai_ros2_wrappers/bdai_ros2_wrappers/tf_listener_wrapper.py) that supports synchronous operation around ROS2’s asynchronous [TF implementation](https://github.com/ros2/rclpy/tree/humble).  Passing it the body and vision frame names causes the wrapper to wait until it sees those frames.  This lets us make sure the robot is started and TF is working before proceeeding.

In order to perform small actions with the robot we use the [SimpleSpotCommander class](../utilities/utilities/simple_spot_commander.py).  This wraps some service clients that talk to services offered by the spot driver.
```python
        self._robot = SimpleSpotCommander()
```

Finally we want to be able to command Spot to do things.  We do this via a [wrapper](https://github.com/bdaiinstitute/ros_utilities/blob/main/bdai_ros2_wrappers/bdai_ros2_wrappers/action_client.py) around the action client that talks to an action server running in the Spot driver (for more information about ROS2 actions see [here](https://docs.ros.org/en/humble/Tutorials/Beginner-CLI-Tools/Understanding-ROS2-Actions/Understanding-ROS2-Actions.html)):
```python
        self._robot_command_client = ActionClientWrapper(RobotCommand, 'robot_command')
```
The wrapper we use here automatically waits for the action server to become available during construction.  It also offers the `send_goal_and_wait` function that we’ll use later.

Before we can walk around, we need to claim the robot’s lease, power it on, and stand it up.  We can do all of these via the spot commander:
```python
    def initialize_robot(self):
        self.get_logger().info('Claiming robot')
        result = self._robot.command('claim')
        if not result.success:
            self.get_logger().error('Unable to claim robot message was ' + result.message)
            return False
        self.get_logger().info('Claimed robot')

        # Stand the robot up.
        self.get_logger().info('Powering robot on')
        result = self._robot.command('power_on')
        if not result.success:
            self.get_logger().error('Unable to power on robot message was ' + result.message)
            return False
        self.get_logger().info('Standing robot up')
        result = self._robot.command('stand')
        if not result.success:
            self.get_logger().error('Robot did not stand message was ' + result.message)
            return False
        self.get_logger().info('Successfully stood up.')
        return True
```

Now we’re ready to walk the robot around!
```python
def walk_forward_with_world_frame_goal(self):
        self.get_logger().info('Walking forward using a world frame goal')
        world_t_robot = self._tf_listener.lookup_a_tform_b(VISION_FRAME_NAME,
                                                           BODY_FRAME_NAME).get_closest_se2_transform()
        world_t_goal = world_t_robot * ROBOT_T_GOAL
```
Goals need to be sent in a static world frame (at least in v3.3…) but we have an offset in the body frame.  Therefore, we first look up the robot’s current position in the world using TF.  We then multiply by the offset we want in the robot frame.  Note that "vision" is Spot’s name for the static world frame it updates using visual odometry.  See [here](https://dev.bostondynamics.com/docs/concepts/geometry_and_frames) for more about Spot’s frames.

Now we construct a goal to send to the Spot driver:
```python
        proto_goal = RobotCommandBuilder.synchro_se2_trajectory_point_command(
            goal_x=world_t_goal.x, goal_y=world_t_goal.y, goal_heading=world_t_goal.angle,
            frame_name=VISION_FRAME_NAME)
        action_goal = RobotCommand.Goal()
        conv.convert_proto_to_bosdyn_msgs_robot_command(proto_goal, action_goal.command)
```
This constructs a protobuf message using the BD built in tools and converts it into a ROS action goal of a type that can be sent by our action client.

Finally we send the goal to the Spot driver:
```python
        self._robot_command_client.send_goal_and_wait(action_goal)
```
Note that `send_goal_and_wait` is a function of our [ActionClientWrapper](https://github.com/bdaiinstitute/ros_utilities/blob/main/bdai_ros2_wrappers/bdai_ros2_wrappers/action_client.py) and not a built in ROS2 function.
