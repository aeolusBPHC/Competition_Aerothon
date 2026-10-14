[[jetson]]

/qr_detection
```

```

/red_zone_detection
```

```

/qr_alignment
```
```

/red_zone_avoid
```
#forbidden cells from red zone mask
```

/exploration_map
```
#track QR code 
#known area + explored/unexplored regions
```

/frontier_manager
```
#drone pose + exploration map + red zone --> exploration goal + new frontiers 
```
/mission_manager
```
#which QR code to go to next + QR alignment and detection and rejection
```

/navFN
```
#navFn / A* + local controller + global, local costmaps + red zone costmap + navigatetopose
```

/nav2_to_sitl_bridge
```

```

/launch
```
#launch/perception.launch.py, launch/navigation.launch.py, #launch/mission.launch.py, launch/phase1.launch.py

```

/configuration
```
config/nav2_params.yaml, config/red_zone_params.yaml, config/mission_params.yaml
```

package.xml or CMakeLists.txt or setup.py
```
```

