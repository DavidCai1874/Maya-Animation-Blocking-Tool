# 如何使用
1. 开启maya
2. 点击右下角唤出代码窗口
<img width="25" alt="Code Editor Icon" src="https://github.com/user-attachments/assets/3db8f240-13cc-4308-8bc9-3ad32263e2a7">


3. 选择python并且粘贴 'animation_blocking_v10.py' 文件中的代码
<img width="677" alt="Python Tab" src="https://github.com/user-attachments/assets/90c3cca5-ac1b-4eab-b2de-7ed437635c0e">


4. 运行
<img width="676" alt="Run Code" src="https://github.com/user-attachments/assets/89ba9ae6-55d6-4dbe-a9c0-b7c715078c7e">

# 工具面板
![{84516F46-B160-4E3B-9D49-61BB17E80505}](https://github.com/user-attachments/assets/40c64b9c-cbc5-4649-a44e-444c5fdf0304)


# 演示视频(Youtube)
[![Watch on YouTube](https://youtu.be/yFxr4EYzWKU/0.jpg)](https://youtu.be/yFxr4EYzWKU)

#### 以防万一
代码在此
```

import maya.cmds as cmds

"""
structure:
1. Attributes
2. Hierachy
3. Frame pick
4. UI
"""

#* ////////////////////////////////////////////// Attributes

ATTRIBUTES = [
    'translateX', 'translateY', 'translateZ',
    'rotateX', 'rotateY', 'rotateZ',
    'scaleX', 'scaleY', 'scaleZ'
]

checkbox_dict = {}

def set_all_checkboxes(state=True):
    for checkbox in checkbox_dict.values():
        cmds.checkBox(checkbox, edit=True, value=state)


#* ///////////////////////////////////////////// Hierachy


all_types = ['group', 'mesh', 'nurbsCurve', 'joint', 'camera', 'light']

type_filter_checkboxes = {}

#called by UI
def update_object_types(*_): #update the types of selected objects
    selected = cmds.ls(selection=True)
    if not selected:
        for typ, cb in type_filter_checkboxes.items():
            cmds.checkBox(cb, edit=True, enable=False)
        return

    found_types = set()
    hierarchy = cmds.listRelatives(selected, allDescendents=True, fullPath=True) or []
    hierarchy += selected

    for obj in hierarchy:
        if not cmds.objExists(obj):
            continue
        node_type = cmds.nodeType(obj)
        shapes = cmds.listRelatives(obj, shapes=True, fullPath=True) or []

        if node_type == 'transform':
            if not shapes:
                found_types.add('group')  # 自定义类型：没有shape的transform
            else:
                for shape in shapes:
                    shape_type = cmds.nodeType(shape)
                    found_types.add(shape_type)
        else:
            found_types.add(node_type)

    for typ, cb in type_filter_checkboxes.items():
        cmds.checkBox(cb, edit=True, enable=(typ in found_types))

def restore_object_types(*_): #restore default state
    for typ, cb in type_filter_checkboxes.items():
        cmds.checkBox(cb, edit=True, value = True,  enable=False)



#called by apply_blocking_to_selected and apply_spline_to_selected
def get_filtered_objects():
    selected = cmds.ls(selection=True, long=True)
    if not selected:
        cmds.warning("Please select at least one object")
        return []

    hierarchy = cmds.listRelatives(selected, allDescendents=True, fullPath=True) or []
    hierarchy += selected

    enabled_types = [typ for typ, cb in type_filter_checkboxes.items()
                     if cmds.checkBox(cb, query=True, value=True)]

    result = []
    for obj in hierarchy:
        if not cmds.objExists(obj):
            continue

        node_type = cmds.nodeType(obj)
        shapes = cmds.listRelatives(obj, shapes=True, fullPath=True) or []

        object_types = []
        if node_type == 'transform':
            if not shapes:
                object_types.append('group')
            else:
                for shape in shapes:
                    object_types.append(cmds.nodeType(shape))
        else:
            object_types.append(node_type)

        if any(t in enabled_types for t in object_types):
            result.append(obj)

    return result


#* /////////////////////////////////// Frame range

frame_range = {'start': 1, 'end': 120}

def update_frame_range(*_):
    start = cmds.intFieldGrp("frameRangeField", query=True, value1=True)
    end = cmds.intFieldGrp("frameRangeField", query=True, value2=True)
    if start >= end:
        cmds.confirmDialog(
            title="Invalid Frame Range",
            message="Start frame must be less than end frame.",
            button=["OK"],
            icon="warning"
        )
        cmds.intFieldGrp("frameRangeField", edit=True, value1=frame_range['start'], value2=frame_range['end'])
        return
    
    frame_range['start'] = start
    frame_range['end'] = end
    cmds.warning(f"Frame range set to: Start = {frame_range['start']}, End = {frame_range['end']}")
    
def set_to_playback_range(*_):
    start = cmds.playbackOptions(query=True, min=True)
    end = cmds.playbackOptions(query=True, max=True)
    frame_range['start'] = int(start)
    frame_range['end'] = int(end)
    cmds.intFieldGrp("frameRangeField", edit=True, value1=frame_range['start'], value2=frame_range['end'])
    cmds.warning(f"Frame range set to playback range: Start = {frame_range['start']}, End = {frame_range['end']}")

def set_to_animation_range(*_):
    start = cmds.playbackOptions(query=True, animationStartTime=True)
    end = cmds.playbackOptions(query=True, animationEndTime=True)
    frame_range['start'] = int(start)
    frame_range['end'] = int(end)
    cmds.intFieldGrp("frameRangeField", edit=True, value1=frame_range['start'], value2=frame_range['end'])
    cmds.warning(f"Frame range set to animation range: Start = {frame_range['start']}, End = {frame_range['end']}")



    
#* /////////////////////////////////// Set Tangent / Spine

##
# Blocking
def apply_blocking_to_selected():
    selected_attrs = [attr for attr, checkbox in checkbox_dict.items()
                      if cmds.checkBox(checkbox, query=True, value=True)]
    if not selected_attrs:
        cmds.confirmDialog(
            title="Invalid Attribute Selection",
            message="Please select at least one attribute.",
            button=["OK"],
            icon="warning"
        )
        return

    for obj in get_filtered_objects():
        for attr in selected_attrs:
            full_attr = f"{obj}.{attr}"
            if cmds.objExists(full_attr):
                create_blocking_animation(full_attr)
                
def create_blocking_animation(attr_full):
    keyframes = cmds.keyframe(attr_full, query=True, timeChange=True)
    if not keyframes:
        return

    keyframes = sorted(set(int(k) for k in keyframes if frame_range['start'] <= k <= frame_range['end']))
    for frame in keyframes:
        cmds.keyTangent(attr_full, time=(frame,), edit=True,
                        inTangentType="stepnext", outTangentType="stepnext")
#end of blocking          
                

##
# Spine
def apply_spline_to_selected():
    selected_attrs = [attr for attr, checkbox in checkbox_dict.items()
                      if cmds.checkBox(checkbox, query=True, value=True)]
    if not selected_attrs:
        cmds.confirmDialog(
            title="Invalid Attribute Selection",
            message="Please select at least one attribute.",
            button=["OK"],
            icon="warning"
        )
        return

    for obj in get_filtered_objects():
        for attr in selected_attrs:
            full_attr = f"{obj}.{attr}"
            if cmds.objExists(full_attr):
                set_spline_tangent(full_attr)
                
def set_spline_tangent(attr_full):
    keyframes = cmds.keyframe(attr_full, query=True, timeChange=True)
    if not keyframes:
        return

    keyframes = sorted(set(int(k) for k in keyframes if frame_range['start'] <= k <= frame_range['end']))
    for frame in keyframes:
        cmds.keyTangent(attr_full, time=(frame,), edit=True,
                        inTangentType="auto", outTangentType="auto")
                


#* /////////////////////////////////// UI Window

def build_blocking_ui():
    if cmds.window("blockingWin", exists=True):
        cmds.deleteUI("blockingWin")

    #Main Window
    cmds.window("blockingWin", title="Animation Blocking Tool", widthHeight=(320, 800))
    cmds.window("blockingWin", edit=True, widthHeight=(350, 550))
    cmds.columnLayout(adjustableColumn=True, rowSpacing=5)


    #attibutes
    cmds.text(label="Select Attributes:")
    cmds.rowColumnLayout(numberOfColumns=3, columnWidth=[(1, 100), (2, 100), (3, 100)])
    for attr in ATTRIBUTES:
        checkbox_dict[attr] = cmds.checkBox(label=attr, value=True)
    cmds.setParent("..")
    #Selections
    cmds.rowLayout(numberOfColumns=2, columnWidth2=(140, 140), adjustableColumn=True)
    cmds.button(label="Select All", command=lambda *_: set_all_checkboxes(True))
    cmds.button(label="Deselect All", command=lambda *_: set_all_checkboxes(False))
    cmds.setParent("..")
    cmds.separator(height=10, style="in")


    # Types (Collapsible Section)
    cmds.frameLayout(label="Object Types Filter", collapsable=True, collapse=True, marginHeight=5, marginWidth=5)
    cmds.text(label="Step 1: Select Hierarchy")
    cmds.rowLayout(numberOfColumns=2, columnWidth2=(140, 140), adjustableColumn=True)
    cmds.button(label="Detect Types in Selection", command=update_object_types)
    cmds.button(label="Restore Types", command=restore_object_types)
    cmds.setParent("..")
    
    cmds.text(label="Step 2: Choose Types to Include:")
    for typ in all_types:
        type_filter_checkboxes[typ] = cmds.checkBox(label=typ, value=True, enable=False)
    cmds.setParent("..")  # Exit frameLayout

    cmds.separator(height=10, style="in")
    
    # Frame Range
    cmds.text(label="Set Frame Range: [Press Enter to Apply Changes]")
    
    
    cmds.intFieldGrp("frameRangeField", numberOfFields=2, label="Start/End", value1= 1, value2= 120,
                     changeCommand=update_frame_range)
    cmds.rowLayout(numberOfColumns=2, adjustableColumn=2)
    cmds.button(label="Set to Playback Range", command=lambda *_: set_to_playback_range())
    cmds.button(label="Set to Animation Range", command=lambda *_: set_to_animation_range())
    cmds.setParent("..")


    
    cmds.separator(height=10, style="in")
    
    # Block / 
    cmds.button(label="Apply Blocking", height=40, command=lambda *_: apply_blocking_to_selected())
    cmds.button(label="Convert to Spline", height=40, command=lambda *_: apply_spline_to_selected())

    cmds.showWindow("blockingWin")






build_blocking_ui()

```

