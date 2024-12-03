- Animals and Children
	- animals locomotion - Solved
	- Gross motor skills vs Fine motor skills
		- Vision issue

- 
- Why Manipulation
	- Because it is hard
		- Humungous Hand
	- Useful
	- separates us from animals (tool use)
- Link between animals and us
	- locomotion -> solved using RL videos
		- https://www.youtube.com/watch?v=iL833P0Vino (deep robotics)
		- Other works
		- RMA and similar
		- Low variation
		- RL
- Manipulation and Vision?
- Can we do manipulation without vision
	- touch based?
- sensors
	- Vision -> RGB, Depth, Pointcloud
	- touch
		- meta, digit, skin etc
- Representations
	- Direct, keypoints
- Ideas/frameworks
	- End to End
	- Intermediate
	- Old School
	- LLM based
	- Liquid NN
- Ongoing
	- Multitask
- Success
	- Imitation learning
	- single policy kind of work now
- Failures
	- Degrade over time
	- Less robust to env changes
	- Robustness require huge data
		- Real vs Sim
	- Delicate/precise/dynamic tasks
		- May be solved using sim
- Sim to Real Gap
- Tools
	- Simulators
	- Real data
	- Teleop
- RL vs Imitation
- Reward, preferemces

Some actual practical cases 
- just pick a object


Taks not attempted yet
- Dish Clearning
- Buttoning a shirt

- General
	- More than vision
	- tactile, force, full hand skin
	- use of full hand (fingers is essential)
- Simulation visuals
	- https://www.youtube.com/watch?v=nz-EklHnN0o













- Start/Intro (why Manipulation is important to solve)
	- History of Robotics 
	- Main challenges in Robotics now
		- Locomotion
		- Manipulation
		- Navigation (not discuss much now)
	- Why I think Manipulation is and important
		- Biological/Evolution Argument
			- Evolution sovled navigation and locomotion earlier then manipulation
			- Animals can do locomotion and navigation 
				- difficult to do manipulation at human scale
			- Link of manipulation with human brain development
				- tools usage
			- Computations
				- Homoculus - hand have large part in brain compare to other
- Why Manipulation is hard
	- Data argument
		- Navigation - Vision data
		- Locomotion - RL (Sim)
			- Human defined Reward funciton
		- Manipulation - No huge data at interent scale
			- RL appraoch - Don't have Simulation with diversity covering all aspects of manipultion
	- Task defintiion
		- Hard to define task in manipulation
		- Language is required
		- Thus RL approach is difficult
			- Difficult to define reward for onion peeling
	- Sense of touch/force become important
- History of robotic manipulation 
	- early tasks
	- before deep leaning
- Era of Deep learning - Recent SOTA approaches
	- RL
	- Imitation Learning
	- Sim to Real
- Open challenges and future directions
	- 












Robotic manipulation has emerged as one of the most complex and transformative challenges in modern robotics, encompassing vision, reinforcement learning (RL), deep learning, and imitation learning. While animal locomotion is largely considered solved through RL-based approaches, as seen in quadrupedal robotics, manipulation remains distinctively harder due to the intricate coordination of gross and fine motor skills, compounded by the complexities of vision-based perception [CITATION]. The human ability to manipulate tools with dexterity separates us from most animals and highlights the unique challenges of robotic manipulation, where achieving similar proficiency remains an open problem [CITATION].

The seminar delves into the critical role of sensory integration in manipulation, contrasting vision-based systems utilizing RGB, depth, and point cloud data with emerging tactile sensing technologies such as Meta’s DIGIT and artificial skin sensors [CITATION]. It explores whether manipulation can be achieved without vision, emphasizing touch-based paradigms and the potential of force feedback for precise, delicate tasks. Representation frameworks are examined, ranging from direct end-to-end learning pipelines to intermediate approaches leveraging keypoints, as well as cutting-edge methods incorporating liquid neural networks and large language models (LLMs) for control and decision-making [CITATION].

Despite significant successes in imitation learning, particularly in single-policy systems capable of performing discrete tasks, current methodologies often degrade over time and struggle with robustness in dynamic or unpredictable environments. Robust manipulation demands massive datasets, and the sim-to-real gap persists as a major hurdle, where simulation fidelity and real-world transfer remain suboptimal [CITATION]. The seminar highlights ongoing research in multi-task learning and RL versus imitation learning paradigms, emphasizing the role of reward structures and preference modeling in enhancing policy generalization.

Practical challenges such as dish cleaning and buttoning a shirt exemplify tasks not yet adequately addressed in the field, underscoring the need for further research into full-hand manipulation, where the coordinated use of fingers and tactile feedback becomes essential [CITATION]. Emerging simulation tools and teleoperation systems are explored, demonstrating their potential in bridging the gap between virtual training environments and real-world deployment [CITATION]. The seminar also examines the limitations of existing approaches, such as the dependence on vast amounts of real-world data and the brittleness of current models under environmental variations, while proposing future directions in sensor fusion, adaptive learning algorithms, and scalable robotic frameworks.

This comprehensive analysis underscores the need for a more holistic understanding of robotic manipulation, integrating vision, touch, and advanced control strategies to push the boundaries of what robots can achieve in complex, real-world scenarios. By addressing the sim-to-real gap, enhancing robustness, and exploring novel task domains, the field is poised to unlock new possibilities in autonomous robotic manipulation.

