### 在virtualenv中运行jupyter

1. 安装jupyter  
`pip install jupyter`

2. 加载venv  
`env\Scripts\activate.bat`  
>注意：此时如果直接运行`jupyter notebook`会报错：  
>`Fatal error in launcher: Unable to create process using '"....\env\scripts\python.exe"  "....\env\Scripts\jupyter-notebook.exe" '`

3. 安装和配置IPykernel  
```
pip install ipykernel
pip uninstall pyzmq  // 必须重装pyzmq, 否则下一步将出错
pip install pyzmq
python -m ipykernel install --user --name py3env --display-name "Python 3 (py3env)"  // 将venv加入jupyter内核列表
```

4. 运行jupyter  
`jupyter notebook`  
此时已经可以启动notebook了；  
修改kernel为我们设定的Python 3 (py3env) 即可。

