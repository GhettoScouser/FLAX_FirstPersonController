# FLAX_FirstPersonController
A First Person Controller for the game engine FLAX. Its well commented so you can read it and figure out what each line does and even modify it to your own needs

## Editor setup
Before you add the script you need some actors in the editor to represent the player:
1. EmptyActor "PlayerContainer"
   - RigidBody "Player"
   - CapsuleCollider "CapsuleCollider"
     - EmptyActor "PlayerHead"
     - Camera "Camera"
It should look like the following image:
![image](https://github.com/user-attachments/assets/685fb5ae-6f1b-4e3b-a8fb-4d542e87b6ee)

This is how the capsule collider is set up:
![image](https://github.com/user-attachments/assets/2ee081f8-9ce0-4ffe-b42e-30b70b7743d1)

This is how the PlayerHead is set up:
![image](https://github.com/user-attachments/assets/96b1d225-9c81-4aeb-aff0-9cbad25748f3)


## Rigibody settings
Set the mass of the player rigidbody to ~100 (Can be changed if needed)
Set the Constraints to: `Lock Rotation X` and `Lock Rotation Y`


## Attaching script
Navigate to the source folder where your scripts are and make a new C# script
Remove any code inside the script and paste the following:
```
using System;
using System.Collections.Generic;
using FlaxEngine;

namespace Game;

/// <summary>
/// PlayerScript Script.
/// </summary>
public class PlayerScript : Script
{
    
    [Header("Player Camera")]
    public float sens;
    public float CameraSmoothing;
    
    
    [Header("Player Movement")]
    public float speed;
    public float SpeedCap;
    public float JumpForce;
    public LayersMask NotPlayer;
    
    
    // Storage
    private RigidBody PlayerRigid;
    private Actor PlayerHead;
    private Actor PlayerContainer;
    private float headHeight;

    private Vector2 PlayerInput;
    
    private Vector2 MouseDelta;
    private Vector2 CameraVector;
    private Transform CameraTransform;
    private float CameraSmoothingFactor;
    
    /// <inheritdoc/>
    public override void OnStart()
    {
        PlayerRigid = Actor as RigidBody; // Get the player
        PlayerHead = PlayerRigid.GetChild("PlayerHead"); // Get the player head (The camera should be a child of this)
        
        // keeping the head as a child of the player body will create a feedback loop when modifying rotations. this splits the head from the body.
        headHeight = PlayerHead.LocalPosition.Y; // Get the height of the head before splitting
        PlayerContainer = PlayerRigid.Parent; // Get player container
        PlayerHead.Parent = PlayerContainer; // This detaches the head from the body and stops a feedback loop when modifying rotations
        
        Screen.CursorVisible = false; // Hide mouse
        Screen.CursorLock = CursorLockMode.Locked; // Lock mouse to center of screen
    }
    
    /// <inheritdoc/>
    public override void OnEnable()
    {
        // Here you can add code that needs to be called when script is enabled (eg. register for events)
    }

    /// <inheritdoc/>
    public override void OnDisable()
    {
        // Here you can add code that needs to be called when script is disabled (eg. unregister from events)
    }

    /// <inheritdoc/>
    public override void OnUpdate()
    {
        /// CAMERA ///
        // Get mouse inputs and store in vector2. Time.DeltaTime to prevent framerate dependant input
        MouseDelta.X = (Input.GetAxis("Mouse Y")* sens);
        MouseDelta.Y = (Input.GetAxis("Mouse X")* sens);
        MouseDelta *= Time.DeltaTime;
        
        // Add MouseDelta inputs to the total Pitch and yaw of the camera
        CameraVector.X = Mathf.Clamp(CameraVector.X + MouseDelta.X, -88f, 88f); // This stops the player from being able to push the vertical view past intended
        CameraVector.Y += MouseDelta.Y;
        
        // While the previous variables were called "CameraVector", we will apply the camera logic to an in game empty representing the player head, then make the actual camera a Child of the head
        PlayerHead.Orientation = Quaternion.Lerp(PlayerHead.Orientation, Quaternion.Euler(CameraVector.X, CameraVector.Y, 0f), CameraSmoothing); // Lerp so we can add mouse smoothing.
        
        
        /// Movement ///
        // Combine the 2 separate input axis into 1 vector2 and Multiply input vector by player transforms so that the Input becomes a local vector
        PlayerInput = new Vector2(Input.GetAxis("Horizontal"), Input.GetAxis("Vertical"));
        var PlayerForce = (Actor.Transform.Forward * PlayerInput.Y + Actor.Transform.Right * PlayerInput.X);
        
        // Floor check. place a raycast sphere under the player. If the sphere is in contact with a surface (that's not the player) then the player is grounded and can move + jump
        // make sure this NotPlayer LayerMask CANNOT interact with the player, that would result in a permanent false positive as half the raycast sphere will be inside the player
        
        var FloorCheckSphere = new BoundingSphere(PlayerRigid.Position + new Vector3(0, 30, 0), 28f); // This is for the debug view of the floor check
        
        if (Physics.SphereCast(PlayerRigid.Position + new Vector3(0, 30, 0), 28, Vector3.Down, 15, NotPlayer))
        {
            DebugDraw.DrawSphere(FloorCheckSphere, Color.Blue, 0f); // This debug lets you visually see the sphere raycast, swapping from red to blue when the player is grounded
            
            // Check if the players current speed is under the speed cap
            if (PlayerRigid.LinearVelocity.Length < SpeedCap)
            {
                // Add force to player
                PlayerRigid.AddForce(PlayerForce * speed * 1000, ForceMode.Force);
            }
            
            // Jump
            if (Input.GetAction("Jump"))
            {
                PlayerRigid.AddForce(new Vector3(0, JumpForce, 0) * 100, ForceMode.Impulse);
            }
        }
        else
        {
            DebugDraw.DrawSphere(FloorCheckSphere, Color.Red, 0f);
        }
        

        /// Synchronise head to body ///
        // Because we seperated the head from the body we need to manually keep them synchronised
        PlayerHead.Position = PlayerRigid.Position + new Vector3(0, headHeight, 0); // Add the head height back in as an offset
        
        // The body of the player (the rigidbody) needs to face the same direction as the head, otherwise player view would not match movement direction
        PlayerRigid.Orientation = Quaternion.Euler(0f, PlayerHead.EulerAngles.Y, 0f); // We only need to spin the player on the Y, keep the other axis at 0 otherwise the player would literally fall over
    }
}
```

## Player inputs
All the inputs in the script should work with the default input file for a new flax project, all besides the jump action.
Go to the root of the project and open the game settings, go to input and make a new action.
- Name "Jump"
- Mode "Press"
- Key "Spacebar"
It should look like the following image:
![image](https://github.com/user-attachments/assets/5adf8d02-136e-4bbe-8c87-3beefb23fac3)


## Player variables
Finally, attatch this script to the Rigidbody "Player" and check the public variables.
You can experiment with these values to get the feel you want, I will provide a good baseline below:
- Sens "8"
- Camera Smoothing "0.7"
- Speed "200"
- Speed cap "300"
- Jump Force "300"
- Layer Mask "Defualt" (Or any layer you want the player to be able to walk on)

## Player physics material
If you find that the player bounces too much when landing or doesnt come to a stop fast enough then you can add a physics materail to the Player CapsuleCollider.
Shown below is the physics material I settle on:
![image](https://github.com/user-attachments/assets/9c61d51c-aec7-47c2-bdfc-d77e5eff4f4d)

