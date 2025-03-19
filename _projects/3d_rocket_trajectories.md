---
layout: page
title: 3D Modeling of Rocket Trajectories into LEO
description: An exercise in numerical simulations, modeling from first principles, and fighting with plotting libraries.
img: assets/img/rocket_traj.gif
importance: 1
category: Modeling & Software
related_publications: true
---


## The Beginning: A Simple 1D Model  
This project started as an exercise in applying intro physics and differential equations.  

To begin, I built a **1D altitude-time simulation**, solving Newton’s second law numerically with Euler's and fourth order Runge-Kutta (RK4) methods. The model included constant thrust, gravity, and quadratic air resistance:

$$
F_{\text{drag}} = -c_d v^2
$$

which led to the velocity and acceleration equations:

$$
v(t) = \frac{dh}{dt}, \quad a(t) = \frac{dv}{dt}
$$

At this stage:  
- The rocket **only moved vertically**, with **constant thrust**.  
- **Euler’s method** was used for numerical integration.  
- **Drag** followed a simple $v^2$ model with no altitude dependence.  

 *Early Output:* 


<div class="row justify-content-sm-center">
    <div class="col-sm-8 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/results.jpg" title="1D Results" class="img-fluid rounded z-depth-1" %}
    </div>
    <div class="col-sm-4 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/failed_sim.jpg" title="1D Results" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    First run of the model, with a failure at $\approx 300 s$.
</div>
The failure was caused by a lack of a conditional drag term in the force equation. As the rocket begins to be accelerated downward by gravity, the negative drag term acts in the same direction as gravity. This causes a rapid breakdown of the model, and a failure of the simulation.




## The Shift to 3D: Full Trajectory Simulation  

Rather than patching the 1D model, I moved directly to a **3D simulation**, where forces could be applied correctly in a **vector-based** framework. This allowed for a more realistic representation of motion, including thrust vectoring, aerodynamic drag, and the effects of Earth's rotation.

### **Why Move to 3D?**
The failure in 1D revealed key limitations:
- **Drag should always oppose velocity**, but in 1D, it was applied in a fixed direction.
- **Gravity acts toward Earth's center**, not just as a constant downward force.
- **Real rockets maneuver**—they don’t follow a simple vertical trajectory.

This made it clear that a **coordinate-based approach** in an **inertial reference frame** was necessary.

---

## **Modeling in a 3D Reference Frame**
To generalize Newton’s Second Law in 3D, I updated the equations to:

$$
m \frac{d\mathbf{v}}{dt} = \mathbf{T} + \mathbf{D} + \mathbf{g}
$$

where:
- **$ \mathbf{T} $** = Thrust force, now directionally controlled.
- **$ \mathbf{D} $** = Drag force, opposing velocity.
- **$ \mathbf{g} $** = Gravity, modeled as a central force.

Instead of tracking just **altitude** $ h(t) $, the model now follows the full **position vector** $ \mathbf{r}(t) $:

$$
\frac{d\mathbf{r}}{dt} = \mathbf{v}, \quad \frac{d\mathbf{v}}{dt} = \frac{1}{m}(\mathbf{T} + \mathbf{D} + \mathbf{g})
$$

This required switching from scalar calculations to **vector operations** and handling transformations between different reference frames.

---

## **Reference Frames & Transformations**
The simulation needed to account for Earth's rotation and how forces behave in different coordinate systems. I implemented transformations between three key frames:

- **East-North-Up (ENU):** Local launch site frame.
- **Earth-Centered, Earth-Fixed (ECEF):** Rotates with Earth.
- **Earth-Centered Inertial (ECI):** Fixed for orbital dynamics.

To switch between these, I used transformation matrices:

### **ECEF to ECI Transformation:**
$$
\mathbf{R}_{\text{ECEF to ECI}}(t) =
\begin{bmatrix}
\cos\Omega_E t & \sin\Omega_E t & 0 \\
-\sin\Omega_E t & \cos\Omega_E t & 0 \\
0 & 0 & 1
\end{bmatrix}
$$

where $ \Omega_E $ is Earth's rotation rate.

This allowed the model to simulate how **Earth’s rotation gives the rocket an initial velocity boost**—a key advantage for real launches.

---

## **Forces Acting on the Rocket**
Once reference frames were established, the full force equation could be formulated as:

$$
m(t) \frac{d\mathbf{v}}{dt} = \mathbf{T} + \mathbf{G} + \mathbf{D}
$$

where:
- **$ \mathbf{T} $ (Thrust):** Engine-generated force.
- **$ \mathbf{G} $ (Gravity):** Modeled as a central force acting toward Earth's center.
- **$ \mathbf{D} $ (Drag):** Atmospheric resistance opposing velocity.

### **Gravity Model**
Gravity follows the inverse-square law:

$$
\mathbf{G} = - \frac{\mu m(t)}{|\mathbf{r}|^3} \mathbf{r}
$$

where:
- $ \mu $ is Earth's standard gravitational parameter.
- $ m(t) $ accounts for mass depletion over time.

### **Drag Model**
Drag is dependent on velocity and altitude:

$$
\mathbf{D} = -\frac{1}{2} \rho(h) C_D A |\mathbf{v}| \mathbf{v}
$$

where:
- $ \rho(h) = \rho_0 e^{-h/H} $ is the atmospheric density model.
- $ C_D $ is the drag coefficient.
- $ A $ is the reference cross-sectional area.

This ensures **drag acts in the opposite direction of velocity** at all times.

---

## **Thrust Vectoring and Gravity Turn**
To execute a **gravity turn**, thrust vectoring was employed, dynamically adjusting the thrust direction based on altitude. The thrust direction was modeled as:

$$
\hat{\mathbf{T}} = \cos\theta \, \hat{\mathbf{r}} + \sin\theta \, \hat{\mathbf{e}}_E
$$

where:
- $ \hat{\mathbf{r}} $ is the radial unit vector.
- $ \hat{\mathbf{e}}_E $ is the eastward unit vector.
- $ \theta $ is the **pitch angle**, evolving throughout the launch.

The pitch angle was defined based on altitude:

$$
\theta =
\begin{cases}
0, & h < h_{\text{start}}, \\
\theta_{\max} \frac{h - h_{\text{start}}}{h_{\text{end}} - h_{\text{start}}}, & h_{\text{start}} \leq h \leq h_{\text{end}}, \\
\theta_{\max}, & h > h_{\text{end}}.
\end{cases}
$$

This allowed the rocket to **naturally transition from vertical ascent into an orbital trajectory**.

---

## **Numerical Integration & RK4 Implementation**
To solve the equations of motion, I transitioned from Euler’s method to a more **stable and accurate** approach: **fourth-order Runge-Kutta (RK4).** This improves numerical precision and prevents divergence in long-duration simulations.

RK4 updates the state vector $ \mathbf{S} $:

$$
\mathbf{S}(t) = \begin{bmatrix} \mathbf{r} \\ \mathbf{v} \\ m \end{bmatrix}
$$

using the iterative scheme:

$$
\mathbf{S}_{n+1} = \mathbf{S}_n + \frac{\Delta t}{6} (k_1 + 2k_2 + 2k_3 + k_4)
$$

where $ k_1, k_2, k_3, k_4 $ are intermediate slopes.

This method allowed the simulation to remain **stable even with complex force interactions**.

---

## **Results: Simulating a Full Trajectory**
With the **3D model complete**, I ran full launch simulations, tracking:
- **Altitude vs. Time**  
- **Velocity Profiles**  
- **Thrust Vectoring Adjustments**  
- **Orbital Insertion Dynamics**

🚀 *First 3D Trajectory Output:* *(Insert 3D trajectory plot here.)*

This confirmed that the **full vector-based model successfully handled launch physics**, including the gravity turn and Earth’s rotation.

---

## **Future Improvements & Next Steps**
While the model works, there’s still room for improvement:
- **Staging Mechanics:** Implementing multi-stage propulsion.
- **More Precise Atmospheric Drag:** Using dynamic coefficient models.
- **Orbital Optimization:** Refining thrust adjustments for LEO insertion.

This project has been a deep dive into **numerical physics, aerospace modeling, and computational methods**, and it’s still evolving.

---

