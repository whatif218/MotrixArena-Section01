在`MotrixLab/motrix_envs/src/motrix_envs`文件夹下解压`vbot_navigation.zip`文件，应该是一个`navigation`文件夹，下面一个有一个`vbot`和`anymal_c`，删除`locomotion`中的`anymal_c`文件夹并修改其下对应的`init.py`文件，改为`from . import go1, go2` 
修改`MotrixLab/motrix_envs/src/motrix_envs`文件夹下的`init.py`，添加`navigation`，也就是改为

    from . import basic, locomotion, manipulation , navigation # noqa: F401


随后将压缩包内的`cfgs.py`文件复制到`MotrixLab/motrix_rl/src/motrix_rl`文件夹下，替换对应的cfg文件

运行可视化：

    uv run scripts/view.py --env vbot_navigation_section01

运行训练：

    uv run scripts/train.py --env vbot_navigation_section01

play需要对应的权重文件，在`MotrixLab`根目录下可能存在`runs`文件夹（如果运行过`train.py`就会自动生成），将其替换为压缩包内的`runs`文件夹即可

运行测试

    uv run scripts/play.py --env vbot_navigation_section01


0215版本更新
添加了第二赛段整体的地图`vbot_navigation_section01`，同时修复了三个阶段的红包等可视化，
具体为在对应的xml文件中添加了

    <model name="section00V" file="0202_V_section00.xml"/>

这个文件，需要注意的是，在`vbot_navigation_section01`这个整体示例中，接触传感器被修改，如果想要使用这个示例，需要自行修改对应的终止和足底接触传感器

0218版本更新
修正了第二赛段第一阶段的碰撞模型
第一赛段的碰撞文件名为
    `<model name="section01V" file="0131_V_section00.xml"/>`
第二赛段第一阶段的碰撞文件名为
    `<model name="section01C" file="0126_C_section01.xml"/>`
容易混淆