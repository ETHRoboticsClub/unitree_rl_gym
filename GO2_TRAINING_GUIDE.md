# Go2 Robot Training Guide

This guide will help you train a reinforcement learning policy for the Unitree Go2 robot dog and test it in Mujoco.

## Quick Start

### 1. Train the Model

Train the Go2 robot to stand and maintain balance:

```bash
python legged_gym/scripts/train.py --task=go2 --headless
```

**Training Parameters:**

- `--task=go2` - Specify the Go2 robot
- `--headless` - Run without GUI for faster training (optional, remove to see visualization)
- `--num_envs=4096` - Number of parallel environments (default: 4096)
- `--max_iterations=1500` - Number of training iterations (default: 1500)

**Expected Training Time:**

- With GPU: ~30-60 minutes for basic locomotion
- Training progress is saved every 50 iterations in `logs/rough_go2/`

### 2. Monitor Training

Watch the training progress in the terminal. Key metrics to monitor:

- **Mean reward**: Should increase over time
- **Episode length**: Should stabilize
- **Policy loss**: Should decrease

### 3. Test the Trained Policy (Play)

After training, test the policy in Isaac Gym:

```bash
python legged_gym/scripts/play.py --task=go2
```

Optional parameters:

- `--load_run=YYYY-MM-DD_HH-MM-SS` - Load specific run
- `--checkpoint=1000` - Load specific checkpoint number

### 4. Export the Policy for Mujoco

The play script automatically exports the policy to TorchScript format. When you run the play command, it will create the exported policy at:

```
logs/rough_go2/exported/policies/policy_1.pt
```

The policy is exported automatically during the play session.

### 5. Test in Mujoco Simulator

Deploy and test your trained policy in Mujoco:

```bash
python deploy/deploy_mujoco/deploy_mujoco.py go2.yaml
```

This will:

- Load your trained policy from `logs/rough_go2/exported/policies/policy_1.pt`
- Simulate the Go2 robot in Mujoco
- Display a visualization window

## Configuration Files

### Training Configuration

Location: `legged_gym/envs/go2/go2_config.py`

Key parameters:

- **Initial height**: 0.42m (standing height)
- **Control type**: Position control (P)
- **Action scale**: 0.25 (limits joint angle changes)
- **Stiffness**: 20 N\*m/rad
- **Damping**: 0.5 N*m*s/rad

### Mujoco Deployment Configuration

Location: `deploy/deploy_mujoco/configs/go2.yaml`

Key parameters:

- **policy_path**: Path to trained policy
- **xml_path**: Path to Go2 Mujoco model
- **kps/kds**: PD controller gains
- **default_angles**: Standing joint angles

## Customization

### For Standing Behavior

If you want the robot to focus on standing still rather than locomotion, modify the reward scales in `legged_gym/envs/go2/go2_config.py`:

```python
class rewards( LeggedRobotCfg.rewards ):
    soft_dof_pos_limit = 0.9
    base_height_target = 0.25
    class scales( LeggedRobotCfg.rewards.scales ):
        # Existing rewards
        torques = -0.0002
        dof_pos_limits = -10.0

        # Add these for standing behavior
        stand_still = -5.0  # Penalize movement
        tracking_lin_vel = 0.0  # Don't reward velocity tracking
        tracking_ang_vel = 0.0  # Don't reward angular velocity
        orientation = -2.0  # Reward upright orientation
        base_height = -10.0  # Reward maintaining target height
```

### Adjusting Training Speed

For faster training (with potential quality tradeoff):

- Reduce `--num_envs` to 2048 (uses less memory)
- Reduce `--max_iterations` to 500-1000

For better quality (slower):

- Increase `--max_iterations` to 3000+
- Keep `--num_envs` at 4096 or higher

## Troubleshooting

### "No module named 'isaacgym'"

Make sure Isaac Gym is installed. See `doc/setup_en.md` for installation instructions.

### Robot Falls Over Immediately

- The policy needs more training iterations
- Check that the initial joint angles are reasonable
- Adjust reward scales to penalize falling

### Policy Doesn't Transfer to Mujoco

- Make sure you exported the policy with `--export` flag
- Verify the policy path in `go2.yaml` is correct
- Check that PD gains (kps/kds) match between training and deployment

### Training is Very Slow

- Add `--headless` flag to disable visualization
- Reduce `--num_envs` if running out of GPU memory
- Make sure you're using CUDA/GPU for training

## File Structure

```
unitree_rl_gym/
├── legged_gym/
│   ├── envs/go2/
│   │   └── go2_config.py          # Training configuration
│   └── scripts/
│       ├── train.py                # Training script
│       └── play.py                 # Testing/export script
├── deploy/
│   └── deploy_mujoco/
│       ├── configs/go2.yaml        # Mujoco deployment config
│       └── deploy_mujoco.py        # Mujoco deployment script
├── resources/robots/go2/
│   ├── scene.xml                   # Mujoco model
│   └── urdf/go2.urdf              # URDF model
└── logs/rough_go2/                 # Training outputs
    ├── YYYY-MM-DD_HH-MM-SS/       # Training run folder
    └── exported/policies/          # Exported policies
```

## Next Steps

1. **Train**: Start with default settings to get a baseline
2. **Evaluate**: Test in Isaac Gym and Mujoco
3. **Iterate**: Adjust rewards and parameters based on behavior
4. **Deploy**: Once satisfied, deploy to real Go2 robot (requires additional setup)

For real robot deployment, see: `deploy/deploy_real/README.md`
