# Sample-and-Search-Return-Project
## Notebook Analysis
1. First, I have uploaded my copy of the notebook in this repository, and all lines of code include comments with explinations.
2. Below are my functions in order to identify obstacles and rocks.
    1. Here is the code for identifying obstacles and a binary image of the rock_img.
        ```python
        #This function will give a binary image for obstacles given an image
        def find_obstacle(img, rgb_thresh=(160, 160, 160)):
            # Create an array of zeros same xy size as img, but single channel
            color_select = np.zeros_like(img[:,:,0])
            # Require that each pixel be below all three threshold values in RGB
            # below_thresh will now contain a boolean array with "True"
            # where threshold was met
            below_thresh = (img[:,:,0] < rgb_thresh[0]) \
                        & (img[:,:,1] < rgb_thresh[1]) \
                        & (img[:,:,2] < rgb_thresh[2])
            # Index the array of zeros with the boolean array and set to 1
            color_select[below_thresh] = 1
            # Return the binary image
            return color_select
        ```    
    2. Here is the code for indentifying rocks and a binary image of the rock_img.
        ```python
        #This function will give the binary immage of rocks given an  image
        def find_rock(img):
    
            # Convert BGR to HSV
            hsv = cv2.cvtColor(img, cv2.COLOR_BGR2HSV)

            # define range of yellow color in HSV    
            lower_yellow = np.array([20,100,100])
            upper_yellow = np.array([100,255,190])

            # Threshold the HSV image to get only yellow colors
            mask = cv2.inRange(hsv, lower_yellow, upper_yellow)
    
        return mask
        ```
3. Below is the code written for the process_image() function.
    ```python
    # Define a function to pass stored images to
    # reading rover position and yaw angle from csv file
    # This function will be used by moviepy to create an output video
    def process_image(img):
        # Example of how to use the Databucket() object defined above
        # to print the current x, y and yaw values 
        # print(data.xpos[data.count], data.ypos[data.count], data.yaw[data.count])

        # TODO: 
        # 1) Define source and destination points for perspective transform
        # The destination box will be 2*dst_size on each side
        dst_size = 5 
        # Set a bottom offset to account for the fact that the bottom of the image 
        # is not the position of the rover but a bit in front of it
        # this is just a rough guess, feel free to change it!
        bottom_offset = 6
        #the source points is a given 1 square meter box in front of the rover, all images are the same size so pixel values can 
        #be constant
        source = source = np.float32([[14, 140], [301 ,140],[200, 96], [118, 96]])
        destination = np.float32([[image.shape[1]/2 - dst_size, image.shape[0] - bottom_offset],
                      [image.shape[1]/2 + dst_size, image.shape[0] - bottom_offset],
                      [image.shape[1]/2 + dst_size, image.shape[0] - 2*dst_size - bottom_offset], 
                      [image.shape[1]/2 - dst_size, image.shape[0] - 2*dst_size - bottom_offset],
                      ])
    
        # 2) Apply perspective transform
        #call perspect transform function, assign it to warped value, and pass in image
        warped = perspect_transform(img, source, destination)

        # 3) Apply color threshold to identify navigable terrain/obstacles/rock samples
        #call color_thresh function to get a binary image for navigable terrian passing in the image
        nav_img = color_thresh(warped)
        #pixel values for obstacles passing in the image
        obstacle_img = find_obstacle(warped)
        #pixel values for the rock samples passing in the image
        rock_img = find_rock(warped)

        # 4) Convert thresholded image pixel values to rover-centric coords
        #rover-centric coords for navigable terrain
        nav_x_pixel, nav_y_pixel = rover_coords(nav_img)
        #rover-centric coords for obstacles
        obs_x_pixel, obs_y_pixel = rover_coords(obstacle_img)
        #rover-centric coords for rocks
        rock_x_pixel, rock_y_pixel = rover_coords(rock_img)


        # 5) Convert rover-centric pixel values to world coords
        #I need to get yaw, xpos and ypos of the rover
        xpos = data.xpos[data.count]
        ypos = data.ypos[data.count]
        yaw = data.yaw[data.count]
        #need to define world size which is for a 200 x 200 world map
        world_size = 200
        scale = 10
        #world coords for navigable terrain
        navigable_x_world, navigable_y_world = pix_to_world(nav_x_pixel, nav_y_pixel, xpos, ypos, yaw, world_size, scale)
        #world coords for obstacles
        obstacle_x_world, obstacle_y_world = pix_to_world(obs_x_pixel, obs_y_pixel, xpos, ypos, yaw, world_size, scale)
        #world coords for rocks
        rock_x_world, rock_y_world = pix_to_world(rock_x_pixel, rock_y_pixel, xpos, ypos, yaw, world_size, scale)

        # 6) Update worldmap (to be displayed on right side of screen)
        data.worldmap[obstacle_y_world, obstacle_x_world, 0] += 1
        data.worldmap[rock_y_world, rock_x_world, 1] += 1
        data.worldmap[navigable_y_world, navigable_x_world, 2] += 1

        # 7) Make a mosaic image, below is some example code
            # First create a blank image (can be whatever shape you like)
        output_image = np.zeros((img.shape[0] + data.worldmap.shape[0], img.shape[1]*2, 3))
            # Next you can populate regions of the image with various output
            # Here I'm putting the original image in the upper left hand corner
        output_image[0:img.shape[0], 0:img.shape[1]] = img

            # Let's create more images to add to the mosaic, first a warped image
        warped = perspect_transform(img, source, destination)
            # Add the warped image in the upper right hand corner
        output_image[0:img.shape[0], img.shape[1]:] = warped

            # Overlay worldmap with ground truth map
        map_add = cv2.addWeighted(data.worldmap, 1, data.ground_truth, 0.5, 0)
            # Flip map overlay so y-axis points upward and add to output_image 
        output_image[img.shape[0]:, 0:data.worldmap.shape[1]] = np.flipud(map_add)


            # Then putting some text over the image
        cv2.putText(output_image,"Populate this image with your analyses to make a video!", (20, 20), 
                    cv2.FONT_HERSHEY_COMPLEX, 0.4, (255, 255, 255), 1)
        data.count += 1 # Keep track of the index in the Databucket()

        return output_image
        ```
4. The video output has been uploaded in the repository. 
  
## Autonomous Navigation and Mapping
1. Perception.py, decision.py, drive_rover.py and supporting_files.py have been uploaded to the repository.
2. Below is the code written for the perception_step().
    ```python
    # Apply the above functions in succession and update the Rover state accordingly
    def perception_step(Rover):
        # Perform perception steps to update Rover()
        # TODO: 
        # NOTE: camera image is coming to you in Rover.img
        image = Rover.img
        # 1) Define source and destination points for perspective transform
        #Define calibration box in source (actual) and destination (desired) coordinates
        # These source and destination points are defined to warp the image
        # to a grid where each 10x10 pixel square represents 1 square meter
        # The destination box will be 2*dst_size on each side
        dst_size = 5 
        # set a bottom offset to account for the fact that the bottom of the image 
        # is not the position of the rover but a bit in front of it
        # this is just a rough guess, feel free to change it!
        bottom_offset = 6
        source = np.float32([[14, 140], [301 ,140],[200, 96], [118, 96]])
        destination = np.float32([[image.shape[1]/2 - dst_size, image.shape[0] - bottom_offset],
                      [image.shape[1]/2 + dst_size, image.shape[0] - bottom_offset],
                      [image.shape[1]/2 + dst_size, image.shape[0] - 2*dst_size - bottom_offset], 
                      [image.shape[1]/2 - dst_size, image.shape[0] - 2*dst_size - bottom_offset],
                      ])

        # 2) Apply perspective transform
        warped = perspect_transform(image, source, destination)

        # 3) Apply color threshold to identify navigable terrain/obstacles/rock samples
        #call color_thresh function to get a binary image for navigable terrian passing in the warped image
        nav_img = color_thresh(warped)
        #pixel values for obstacles passing in the warped image
        obstacle_img = find_obstacle(warped)
        #pixel values for the rock samples passing in the warped image
        rock_img = find_rock(warped)

        # 4) Update Rover.vision_image (this will be displayed on left side of screen)
        Rover.vision_image[:,:,0] = obstacle_img * 255
        Rover.vision_image[:,:,1] = rock_img * 255
        Rover.vision_image[:,:,2] = nav_img * 255

        # 5) Convert map image pixel values to rover-centric coords
        #rover-centric coords for navigable terrain
        nav_x_pixel, nav_y_pixel = rover_coords(nav_img)
        #rover-centric coords for obstacles
        obs_x_pixel, obs_y_pixel = rover_coords(obstacle_img)
        #rover-centric coords for rocks
        rock_x_pixel, rock_y_pixel = rover_coords(rock_img)

        # 6) Convert rover-centric pixel values to world coordinates
        #I need to get yaw, xpos and ypos of the rover
        xpos, ypos = Rover.pos
        yaw = Rover.yaw
        #need to define world size which is for a 200 x 200 world map
        world_size = 200
        #scale defined here
        scale = 10
        #world coords for navigable terrain
        navigable_x_world, navigable_y_world = pix_to_world(nav_x_pixel, nav_y_pixel, xpos, ypos, yaw, world_size, scale)
        #world coords for obstacles
        obstacle_x_world, obstacle_y_world = pix_to_world(obs_x_pixel, obs_y_pixel, xpos, ypos, yaw, world_size, scale)
        #world coords for rocks
        rock_x_world, rock_y_world = pix_to_world(rock_x_pixel, rock_y_pixel, xpos, ypos, yaw, world_size, scale)

        # 7) Update Rover worldmap (to be displayed on right side of screen)
        #Only update the worldmap as long as the roll and pitch angles are almost 0 in order to increase fidelity
        if Rover.roll > 359.8 or Rover.roll < 0.2 or Rover.pitch > 359.8 or Rover.pitch < 0.2:
            Rover.worldmap[obstacle_y_world, obstacle_x_world, 0] += 1
            Rover.worldmap[rock_y_world, rock_x_world, 1] += 1
            Rover.worldmap[navigable_y_world, navigable_x_world, 2] += 1

        #Rover.worldmap[obstacle_y_world, obstacle_x_world, 0] += 1
        #Rover.worldmap[rock_y_world, rock_x_world, 1] += 1
        #Rover.worldmap[navigable_y_world, navigable_x_world, 2] += 1

        # 8) Convert rover-centric pixel positions to polar coordinates
        rover_centric_pixel_distances, rover_centric_angles = to_polar_coords(nav_x_pixel, nav_y_pixel)
        # Update Rover pixel distances and angles
        Rover.nav_dists = rover_centric_pixel_distances
        Rover.nav_angles = rover_centric_angles

        return Rover
        ```
3. Below is the code written for decision_step()
    ```python
    # This is where you can build a decision tree for determining throttle, brake and steer 
    # commands based on the output of the perception_step() function
    def decision_step(Rover):

        # Implement conditionals to decide what to do given perception data
        # Here you're all set up with some basic functionality but you'll need to
        # improve on this decision tree to do a good job of navigating autonomously!

        # Example:
        # Check if we have vision data to make decisions with
        if Rover.nav_angles is not None:
            # Check for Rover.mode status
            if Rover.mode == 'forward': 
                # Check the extent of navigable terrain
                if len(Rover.nav_angles) >= Rover.stop_forward:  
                    # If mode is forward, navigable terrain looks good 
                    # and velocity is below max, then throttle 
                    if Rover.vel < Rover.max_vel:
                        # Set throttle value to throttle setting
                        Rover.throttle = Rover.throttle_set
                    else: # Else coast
                        Rover.throttle = 0
                    Rover.brake = 0
                    # Set steering to average angle clipped to the range +/- 15
                    Rover.steer = np.clip(np.mean(Rover.nav_angles * 180/np.pi), -15, 15)
                # If there's a lack of navigable terrain pixels then go to 'stop' mode
                elif len(Rover.nav_angles) < Rover.stop_forward:
                        # Set mode to "stop" and hit the brakes!
                        Rover.throttle = 0
                        # Set brake to stored brake value
                        Rover.brake = Rover.brake_set
                        Rover.steer = 0
                        Rover.mode = 'stop'

            # If we're already in "stop" mode then make different decisions
            elif Rover.mode == 'stop':
                # If we're in stop mode but still moving keep braking
                if Rover.vel > 0.2:
                    Rover.throttle = 0
                    Rover.brake = Rover.brake_set
                    Rover.steer = 0
                # If we're not moving (vel < 0.2) then do something else
                elif Rover.vel <= 0.2:
                    # Now we're stopped and we have vision data to see if there's a path forward
                    if len(Rover.nav_angles) < Rover.go_forward:
                        Rover.throttle = 0
                        # Release the brake to allow turning
                        Rover.brake = 0
                        # Turn range is +/- 15 degrees, when stopped the next line will induce 4-wheel turning
                        Rover.steer = -15 # Could be more clever here about which way to turn
                    # If we're stopped but see sufficient navigable terrain in front then go!
                    if len(Rover.nav_angles) >= Rover.go_forward:
                        # Set throttle back to stored value
                        Rover.throttle = Rover.throttle_set
                        # Release the brake
                        Rover.brake = 0
                        # Set steer to mean angle
                        Rover.steer = np.clip(np.mean(Rover.nav_angles * 180/np.pi), -15, 15)
                        Rover.mode = 'forward'
        # Just to make the rover do something 
        # even if no modifications have been made to the code
        else:
            Rover.throttle = Rover.throttle_set
            Rover.steer = 0
            Rover.brake = 0

        # If in a state where want to pickup a rock send pickup command
        if Rover.near_sample and Rover.vel == 0 and not Rover.picking_up:
            Rover.send_pickup = True

        return Rover
        ```

4. NOTE: Simulator Settings
    1. Resolution was 1152 x 864 for Windows OS
    2. Graphics Quality was Fantastic
    3. Frames per Second was default
5. Below are the metrics that the rover was able to accomplish.
    
