Project: Highway Driving(Path Planning)
---

[![Udacity - Self-Driving Car NanoDegree](https://s3.amazonaws.com/udacity-sdc/github/shield-carnd.svg)](http://www.udacity.com/drive)

|<img src="data/images/final-output.gif" width="500" height="300" />|
|----------------------------------|
|[Running instructions](https://youtu.be/SLMwKynQ9MU) |

### Introduction
*The goal of this project is to build a path planner that creates smooth, safe trajectories for the car to follow. The highway track has other vehicles, all going different speeds, but approximately obeying the 50 MPH speed limit.*

*The car transmits its location, along with its sensor fusion data, which estimates the location of all the vehicles on the same side of the road.*

### The code model for generating paths
Path planning implementation is coded in the `path_planner.cpp` with the method name `planPath`.  Following is the implementation of `planPath` method.


```cpp
std::pair<std::vector<double>, std::vector<double >> PathPlanner::planPath(
        const path_planning::SimulatorRequest &simReqData)
{
    // Update trajectory history
    updateTrajectoryHistory(simReqData);
    // Get the lane speeds
    std::array<double, 3> laneSpeeds = getLaneSpeeds(simReqData.mainCar, simReqData.otherCars);
    // Scheduling the lane's changes
    scheduleLaneChange(simReqData.mainCar, laneSpeeds, simReqData.otherCars);
    // generate Spiline x and y trajectories
    std::pair<std::vector<double>, std::vector<double>> xy_trajectories =
            generateTrajectorySplines(simReqData.mainCar, laneSpeeds[m_targetLaneIndex], simReqData.previous_path_x,
                                      simReqData.previous_path_y);
    m_lastX = xy_trajectories.first;
    m_lastY = xy_trajectories.second;
    return xy_trajectories;
}

```

Following steps were followed to implement a path planning solution.

##### Step01
Added simulator incommig `x,y` coordinates and updated previusly used some `x,y` points using a double ended queue(`deque`).

```cpp
void PathPlanner::updateTrajectoryHistory(const path_planning::SimulatorRequest &simReqData)
{
    auto executedCommands = m_lastX.size() - simReqData.previous_path_x.size();

    for (auto itr = m_lastX.begin(); itr != m_lastX.begin() + executedCommands; ++itr)
    {
        m_historyMainX.push_front(*itr);
    }
    if (m_historyMainX.size() > TRAJECTORY_HISTORY_LENGTH)
    {
        m_historyMainX.resize(TRAJECTORY_HISTORY_LENGTH);
    }

    for (auto itr = m_lastY.begin(); itr != m_lastY.begin() + executedCommands; ++itr)
    {
        m_historyMainY.push_front(*itr);
    }

    if (m_historyMainY.size() > TRAJECTORY_HISTORY_LENGTH)
    {
        m_historyMainY.resize(TRAJECTORY_HISTORY_LENGTH);
    }
}
```

##### Step02
Check if a lane change is needed and change the target lane accodingly.

```cpp
void PathPlanner::scheduleLaneChange(const path_planning::MainCar &mainCar, const std::array<double, 3> &laneSpeeds,
                                     const std::vector<path_planning::OtherCar> &sensorFusions)
{
    if (std::abs(mainCar.d - m_targetLaneD) > D_LIMIT_FOR_LANE_CHANGE_PENALTY)
    {
        m_laneChangeDelay = LANE_CHANGE_PENALTY;
    }
    else if (m_laneChangeDelay != 0)
    {
        --m_laneChangeDelay;
    }
    else
    {
        if (m_targetLaneD == D_LEFT_LANE && laneSpeeds[1] - LANE_CHANGE_COST > laneSpeeds[0]
            && !isLaneBlocked(D_MIDDLE_LANE, mainCar, sensorFusions))
        {
            m_targetLaneD = D_MIDDLE_LANE;
            m_targetLaneIndex = 1;
            m_laneChangeDelay = LANE_CHANGE_PENALTY;
        }
        else if (m_targetLaneD == D_RIGHT_LANE && laneSpeeds[1] - LANE_CHANGE_COST > laneSpeeds[2]
                 && !isLaneBlocked(D_MIDDLE_LANE, mainCar, sensorFusions))
        {
            m_targetLaneD = D_MIDDLE_LANE;
            m_targetLaneIndex = 1;
            m_laneChangeDelay = LANE_CHANGE_PENALTY;
        }
        else if (m_targetLaneD == D_MIDDLE_LANE &&
                 (laneSpeeds[0] - LANE_CHANGE_COST > laneSpeeds[1] || laneSpeeds[2] - LANE_CHANGE_COST > laneSpeeds[1]))
        {
            if (laneSpeeds[0] > laneSpeeds[2] && !isLaneBlocked(D_LEFT_LANE, mainCar, sensorFusions))
            {
                m_targetLaneD = D_LEFT_LANE;
                m_targetLaneIndex = 0;
                m_laneChangeDelay = LANE_CHANGE_PENALTY;
            }
            else if (!isLaneBlocked(D_RIGHT_LANE, mainCar, sensorFusions))
            {
                m_targetLaneD = D_RIGHT_LANE;
                m_targetLaneIndex = 2;
                m_laneChangeDelay = LANE_CHANGE_PENALTY;
            }
        }
    }
}
```
