# Import Fusion360 Model to Gazebo

This guide provides instructions for setting up the environment to import models from Fusion 360 into Gazebo. The conversion process relies on a tool in ROS 2, so ROS 2 must also be installed.

This setup uses Ubuntu Noble 24.04 + Gazebo Harmonic + ROS 2 Jazzy Jalisco, but other Ubuntu and ROS 2 versions should work. Gazebo Harmonic is recommended for  compatibility. For issues with other versions of Gazebo, see [Troubleshooting in Gazebo](#5-troubleshooting-in-gazebo) at the end.



## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Export Fusion360 Model](#2-export-fusion360-model)
3. [Import to Gazebo](#3-import-to-gazebo)
4. [Fusion Model Settings](#4-fusion-model-settings)
5. [Troubleshooting in Gazebo](#5-troubleshooting-in-gazebo)



## 1 Prerequisites

### Installing ROS 2 and Gazebo

1. **Check your Ubuntu version:**

```shell
lsb_release -a
```

2. **Identify the recommended versions for your system:**

- Check compatible Gazebo versions: `https://gazebosim.org/docs/fortress/getstarted/`
- Refer to ROS installation recommendations: `https://gazebosim.org/docs/fortress/ros_installation/`

3. **Install ROS 2 :**

- If you are intalling ROS 2 jazzy, use:

`https://docs.ros.org/en/jazzy/Installation/Alternatives/Ubuntu-Development-Setup.html`

- For other ROS 2 versions, refer to the general ROS 2 installation documentation:

  `https://docs.ros.org/en/`

4. **Install Gazebo Harmonic:**

- For the recommended Gazebo version, see the Gazebo Harmonic Installation Guide: `https://gazebosim.org/docs/harmonic/install_ubuntu/`

### Environment Configuration

Configure your shell to automatically source the ROS 2 environment each time you open a terminal:

```shell
# For Zsh users:
echo "source /opt/ros/${ROS_DISTRO}/setup.zsh" >> ~/.zshrc
source ~/.zshrc

# For Bash users:
echo "source /opt/ros/${ROS_DISTRO}/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

### Install the xacro Package

The `xacro` package, part of ROS 2, is necessary for converting files from `.xacro` to `.urdf` format.

```
sudo apt install ros-${ROS_DISTRO}-xacro
```



## 2 Export Fusion360 Model

This section is based on a fork from https://github.com/SpaceMaster85/fusion2urdf. The original project is no longer maintained, so I have updated it to be compatible with the latest version of Fusion 360. 

### Installation

Clone this repo

```shell
 git clone git@github.com:YanningDai/Import_Fusion_Model_to_Gazebo.git
```

**Note:** By default, this setup uses ROS 2; however, if you need ROS 1, set the variable `ros_version_is_ros1` to `True` in the `Import_Fusion_Model_to_Gazebo/URDF_Exporter.py` file.

Run the following command in your shell:

##### - Windows (In PowerShell)

```shell
cd Import_Fusion_Model_to_Gazebo
Copy-Item ".\URDF_Exporter\" -Destination "${env:APPDATA}\Autodesk\Autodesk Fusion 360\API\Scripts\" -Recurse
```

##### - macOS (In bash or zsh)

```shell
cd Import_Fusion_Model_to_Gazebo
cp -r ./URDF_Exporter "$HOME/Library/Application Support/Autodesk/Autodesk Fusion 360/API/Scripts/"
```

### Model Export

- **Important:** Before exporting, check your model settings in [Fusion Model Settings](#4-fusion-model-settings).

- Once your model is ready, go to **ADD-INS** in Fusion 360 and select **URDF_Exporter**.

- Run the script and wait for the process to complete (this may take a few seconds or minutes). A folder dialog will then appear—select your desired location for the output files.

- The script will create a folder named after your Fusion 360 model in lowercase, followed by "_description". For example, "robot_description".

  

## 3 Import to Gazebo

Convert the xacro file to SDF format and view it in Gazebo.

```shell
# Navigate to an empty project folder for the output .sdf file
cd /path/to/empty/folder
mkdir src

# Copy files from Fusion 360 output to the src folder
# Replace `robot` with your Fusion360 model's name
export MODEL_NAME=robot
scp -r "/path/to/fusion360/output/${MODEL_NAME}_description" src 

# Build xacro
colcon build
source install/setup.bash  # in zsh: source install/setup.zsh

# Convert xacro -> urdf -> sdf
xacro src/${MODEL_NAME}_description/urdf/${MODEL_NAME}.xacro > src/${MODEL_NAME}_description/urdf/${MODEL_NAME}.urdf
gz sdf -p src/${MODEL_NAME}_description/urdf/${MODEL_NAME}.urdf > src/${MODEL_NAME}_description/urdf/${MODEL_NAME}.sdf

# Then, the generated `.sdf` file and the `meshes` folder (under src/${MODEL_NAME}_description) are what you need for gazebo.

# Set up the Gazebo resource path and preview in Gazebo
export GZ_SIM_RESOURCE_PATH=$(pwd)/src/
gz sim src/${MODEL_NAME}_description/urdf/${MODEL_NAME}.sdf
```

**Note**:

1. Replace `/path/to/empty/folder` with the path to an empty directory for your project.
2. Replace `/path/to/fusion360/output` with the path of your Fusion 360 export files.
3. Update `robot` with the name of your Fusion360 model.



## 4 Fusion Model Settings

**Links**

- Ensure all links are set as components in Fusion 360 by creating corresponding components.

**Joint Constraints**

- Supported Joint Types: Only "Rigid," "Slider," and "Revolute" joints are supported.
- **Parent links** should be set as Component2 when defining a joint. Setting "base_link" as Component1, for instance, will cause a "KeyError: base_link__1" error.

**Export Limitations**

- Nested components are exported as a single STL file; only root-level joints are processed, while joints in nested or lower-level components are ignored.
- Only components and bodies with active light bulbs are exported; those with deactivated light bulbs are ignored. Components connected by deactivated joints will not be exported.

**Colors**

- Colors defined at the root level of components are retained and included in the URDF file’s material definitions.

**Troubleshooting**

- Occasionally, the script may export an abnormal URDF without error messages. If this occurs, try redefining the joints and running the export again.

For more details and examples, see:  https://github.com/SpaceMaster85/fusion2urdf



## 5 Troubleshooting in Gazebo

1. If `gz sim` loads the SDF model but doesn’t display it, try setting the mesh scale to `1:1:1` as it may be too small.

2. If the SDF file can’t find the model path, it could be due to a version mismatch. Use `gz sim -h` to check environment variables, and update `export GZ_SIM_RESOURCE_PATH` to match the model path variable in your version. Some Gazebo versions use `GAZEBO_MODEL_PATH` as the path for models.

3. Materials aren’t set, so `inertial matrix` and `mass` parameters might be incorrect, but this won’t affect the display.

4. If the Gazebo world doesn’t update and remains in a previous state, clear the Gazebo cache with:

   ```shell
   pkill -f gz
   ps aux | grep gz
   ```





