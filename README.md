# Real2Sim

- [Real2Sim](#real2sim)
  - [Installation](#installation)
  - [Reproducing Real-to-Sim Evaluation Results for Google Robot and WidowX](#reproducing-real-to-sim-evaluation-results-for-google-robot-and-widowx)
  - [Adding New Real-to-Sim Evaluation Environments, Robots, and Policies](#adding-new-real-to-sim-evaluation-environments-robots-and-policies)
    - [Adding New Robots](#adding-new-robots)
    - [Adding New Environments](#adding-new-environments)
    - [Adding New Policies](#adding-new-policies)
    - [Appendix](#appendix)
      - [SAPIEN viewer controls](#sapien-viewer-controls)


## Installation

Create an anaconda environment: 
```
conda create -n real2sim python=3.9
```

Clone this repo:
```
git clone https://github.com/xuanlinli17/Real2Sim --recurse-submodules
```

Install ManiSkill2 and its dependencies:
```
cd {this_repo}/ManiSkill2_real2sim
pip install -e .
```

Install this package:
```
cd {this_repo}
pip install tensorflow==2.15.0
pip install -e .
pip install tensorflow[and-cuda]
```

Download RT-1 Checkpoint:
```
# First, install gsutil following https://cloud.google.com/storage/docs/gsutil_install

# RT-1-X
cd {this_repo}
gsutil -m cp -r gs://gdm-robotics-open-x-embodiment/open_x_embodiment_and_rt_x_oss/rt_1_x_tf_trained_for_002272480_step.zip .
unzip rt_1_x_tf_trained_for_002272480_step.zip
rm rt_1_x_tf_trained_for_002272480_step.zip

# RT-1-Converged
Download from https://drive.google.com/drive/folders/1pdHYzgNQqinEv0sXlKpL3ZDr2-eDFebQ

# RT-1-15%
Download from https://drive.google.com/drive/folders/1nzOfnyNzxKkr3aXj3kqekfXdxAPU15aY

# RT-1-Begin
Download from https://drive.google.com/drive/folders/19xWAJR9EGX86zN9LfgYSvKj27t_4kNry
```

Install Octo:
```
cd {this_repo}/octo
pip install --upgrade "jax[cuda11_pip]" -f https://storage.googleapis.com/jax-releases/jax_cuda_releases.html
pip install -e .
```

Install simulated annealing utils for system identification:
```
pip install git+https://github.com/nathanrooy/simulated-annealing
```


## Reproducing Real-to-Sim Evaluation Results for Google Robot and WidowX



## Adding New Real-to-Sim Evaluation Environments, Robots, and Policies

Below we provide a step-by-step guide to add new real-to-sim evaluation environments, robots, and policies.

### Adding New Robots

If you are adding a new robot, perform the following steps:

1. Add your robot urdf to `ManiSkill2_real2sim/mani_skill2/assets/descriptions`. Then, create a test robot script in `tools/robot_object_visualization` to visualize the robot in the SAPIEN viewer. You can use the existing scripts as a reference by changing the loaded urdf path and the initial joint positions (qpos).
   - For SAPIEN viewer control, see [here](#sapien-viewer-controls)

2. Create a new robot agent implementation in `ManiSkill2_real2sim/mani_skill2/agents/robots` (which inherit `ManiSkill2_real2sim/mani_skill2/agents/base_agent.py`). You can use the existing robot implementations as a reference.
   - Add a set of controller configurations for the robot arm and the robot gripper. See more controller details in `ManiSkill2_real2sim/mani_skill2/agents/controllers/` and their base class `ManiSkill2_real2sim/mani_skill2/agents/base_controller.py`. You can also add more controller implementations there. 
     - Other relevant functions are the `step_action` function in `ManiSkill2_real2sim/mani_skill2/envs/sapien_env.py` and the `set_action`, `before_simulation_step` functions in `ManiSkill2_real2sim/mani_skill2/agents/base_agent.py`.
   - Add dummy stiffness and damping controller parameters; we will do system identification later.
   - Add cameras to the robot; camera poses are with respect to the link on the robot specified by `actor_uid`. In SAPIEN, camera pose follows ROS convention, i.e., x forward, y left, z up.

3. Perform system identification for the robot.
   - First, create a system identification dataset. If you have an existing tensorflow dataset, you can create a subset of trajectories for system identification by modifying `tools/sysid/prepare_sysid_dataset.py`. In other cases, you can create a system identification subset by following the saved pickle file format in `tools/sysid/prepare_sysid_dataset.py` and having the necessary keys in each step of a trajectory.
   - Next, perform system identification using the dataset. You can modify the existing system identification script `tools/sysid/sysid.py`. The script uses simulated annealing algorithm to find better stiffness and damping parameters from the initialization parameters. Examine the system identification logs using `tools/sysid/analyze_sysid_results.py`, and use the best parameters to initialize the next round of system identification with reduced parameter search range. After multiple rounds of system identification, you can then use the best parameters to update the robot agent implementation.

### Adding New Environments

SAPIEN uses an axis convention of x forward, y left, z up. 

4. Add new object assets to `ManiSkill2_real2sim/data/custom`. 
   - Example object assets are in `ManiSkill2_real2sim/data/custom`. Each object asset contains two components:
     - Visual mesh (a `textured.dae` file combined with its corresponding `.png` texture files, or a single `textured.glb` file)
     - Collision mesh (`collision.obj`). The collision mesh should be watertight and convex.
   - After adding `collision.obj`, if the collision shape is not yet convex, use `tools/coacd_process_mesh.py` to obtain a convex collision mesh.
   - Use `tools/robot_object_visualization/test_object.py` to visualize the object in the SAPIEN viewer. You can click on an object / object link and press "show / hide" on the right panel to show / hide its collision mesh. 
   - For SAPIEN viewer control, see [here](#sapien-viewer-controls)

<details>
<summary>**Notes on modifying and exporting objects from Blender**: </summary>

- If you have an object file opened in Blender, assuming that an object is modeled with y-forward and z-up convention (e.g., the longest side of object is along the +y axis and the shortest side is along the +z axis), you can export the object as `textured.dae` / `textured.glb` for the textured mesh and `collision.obj` for the collision mesh with `x forward, z up` as the output option in Blender. In this case, when you load the object in SAPIEN, the x-axis will become the longest side while the z aixs will become the shortest side. This is because the Blender axis convention is y forward, z up, while the SAPIEN axis convention is x forward, z up.

- When modeling objects in Blender, it is recommended to clear its parents (`alt+p > clear parent`) and move the object to the origin (`object > set origin > geometry to origin`). Before exporting objects, set them to have a unit transformation `object > apply > all transforms`. When loading and modifying objects in Blender, first select the object, press `N` to toggle object property channel, set the object rotation to euler mode with all 0s, then start modifying the object, and export objects with the forementioned conventions.
  
</details>

5. Add custom simulation scene backgrounds to `ManiSkill2_real2sim/data/hab2_bench_assets/stages`.
   - In our environments, scene backgrounds are loaded in the `_load_arena_helper` function in `ManiSkill2_real2sim/mani_skill2/envs/custom_scenes/base_env.py`. The existing scenes use the Habitat convention (y-axis up).

<details>
<summary>**Notes on Blender:** </summary>

You can export the `.glb` scenes from Blender. Pay attention to the axis convention. When we modify the ReplicaCAD scenes in Blender, we load and save the scenes using the default Blender axis settings (you can restore these default settings through `Operator Presets > Restore Operator Presets` under the load / save window).
</details>
   
6. If you adopt our visual-matching ("greenscreen") evaluation setup, add the overlay background image (with the robot and interactable objects removed through impainting) to `ManiSkill2_real2sim/data/real_impainting`.
   - We use https://cleanup.pictures/ to remove the robot and the interactable objects from the real images.

7. Add a new environment to `ManiSkill2_real2sim/mani_skill2/envs/custom_scenes`. You can use the existing environments as a reference. 
   - The environment `reset` function first assesses whether to reconfigure the environment (if so, then we call the `reconfigure` function in `ManiSkill2_real2sim/mani_skill2/envs/sapien_env.py` to load scenes, robot agents, objects, cameras, and lightings). It then calls the `initialize_episode` function to initialize the loaded robots and objects.

8. Test your environment using `ManiSkill2_real2sim/mani_skill2/examples/demo_manual_control_custom_envs.py`. See the script for more details.
   - You can set different `env_reset_options` to test different environment configurations.


### Adding New Policies

9. 








### Appendix

#### SAPIEN viewer controls

- In the viewer, click on an object / articulated object link and press "f" to focus on it. 
- Use right mouse button to rotate; middle-mouse-button + shift to translate. Scroll the middle mouse button to zoom in / out (if you press shift when you scroll, you can zoom in / out slower).
- Under "scene hierarchy" on the bottom left, you can select different actors and articulation links based on their names. 
- When an articulated object is selected (e.g., robot), then under "articulation" on the bottom right, you can move the scrollbar to change each of its joint positions / angles. 
- Press "pause" on the top left to pause the simulation. 
- Press "g" to grab object; "g" + "x"/"y"/"z" to move object along x/y/z axis.




```
Make mesh a (local) convex hull to reduce "slipping" behaviors
```


**Scripts**

See `scripts/rt1_pick_coke_can_eval.sh`




real2sim/main_inference.py
real2sim/utils/env/env_builder.py
real2sim/utils/env/observation_utils.py
real2sim/tools/sysid/sysid.py
real2sim/tools/robot_object_visualization

tcp = tool center point of end-effector