classDiagram
    direction LR

    %% --- KLASSEN-DEFINITIONEN ---

    class Body {
        +String name
        +Float mass
        +Float radius
        +NumPy_Array_3D position
        +NumPy_Array_3D velocity
        +String color
        +__init__(...)
        +update_velocity(force, dt)
        +update_position(dt)
        +get_momentum()
    }

    class Simulation {
        +List bodies
        +Float G
        +Float time_step
        +Dict history
        +add_body(body)
        +calculate_gravity()
        +check_collisions()
        +merge_bodies(body1, body2)
        +step()
        +run(total_time)
    }

    class Visualizer {
        +Simulation Simulation
        +Float body_scale
        +animate()
    }

    class ScenarioController {
        +Float initial_distance
        +Float speed_magnitude
        +String target_type
        +__init__(speed_magnitude, target_type)
        +calculate_geometry()
        +calculate_velocity_vector(start_pos, target_pos)
        +setup_theia()
    }


    %% --- BEZIEHUNGEN ---

    Simulation "1" --> "*" Body : besitzt / verwaltet
    ScenarioController ..> Simulation : konfiguriert
    Visualizer "1" --> "1" Simulation : visualisiert
    Visualizer ..> Body : nutzt Daten von
