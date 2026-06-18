现象：
header HBoxContainer 下有一个   70px
    VBoxContainer   70px
        HBoxContainer   16px
            Label   默认最小23px font-size = 16px ,默认后会把HBoxContainer撑大到23px
        BoxContainer 16px
            ProgressBar 默认最小23px font-size = 16px ,默认后会把HBoxContainer撑大到23px

        BoxContainer 16px
        BoxContainer 16px

解决方法：
    必须把带font的相关内容的，把他们的字体大小从默认的16px，下调到合适的大小，才不会接着撑大父元素Container的高度