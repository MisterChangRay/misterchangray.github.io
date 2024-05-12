---
layout: post
title:  "c#操作其他进程的SysListView32"
date:   2024-05-13 01:29:20 +0800
categories:
      - c#
      - win32
tags:
      - SysListView32
---

有些时候，开发pc端的时候想读取或操作其他进程的 Listview, 一般来说spy++查看 类名为:`SysListView32`

![image](https://github.com/MisterChangRay/misterchangray.github.io/assets/16421384/fbc7ff41-8cb1-4709-9b62-de3d73ad6304)


先贴上官方文档：
[选中消息事件](https://learn.microsoft.com/zh-cn/windows/win32/controls/lvm-setitemstate)

[消息定义](https://github.com/tpn/winsdk-10/blob/master/Include/10.0.16299.0/um/CommCtrl.h)



下面代码实现了获取第三方的 Listview 数据，和选中功能。 分别位于selectItem2/ getText 两个方法。

主要留存下代码，方便下次使用



希望能帮助到大家:

```c#
using System;
using System.Collections.Generic;
using System.Diagnostics;
using System.Linq;
using System.Reflection;
using System.Reflection.Emit;
using System.Runtime.InteropServices;
using System.Text;
using System.Threading.Tasks;

namespace spiderClient
{
    internal class ListUtil
    {

        int hwnd;   //窗口句柄
        int process;//进程句柄
        IntPtr pointer;
        private const uint LVM_FIRST = 0x1000;
        private const uint LVM_SETSELECTEDCOLUMN = LVM_FIRST + 140;
        
        private const uint LVM_GETHEADER = LVM_FIRST + 31;
        private const uint LVM_SETSELECTIONMARK = LVM_FIRST + 67;
        private const uint LVM_SETITEMSTATE = LVM_FIRST + 43;
        

        private const uint LVM_GETITEMCOUNT = LVM_FIRST + 4;//获取列表行数
        private const uint LVM_GETITEMTEXT = LVM_FIRST + 45;//获取列表内的内容
        private const uint LVM_GETITEMW = LVM_FIRST + 75;

        private const uint HDM_GETITEMCOUNT = 0x1200;//获取列表列数

        private const uint PROCESS_VM_OPERATION = 0x0008;//允许函数VirtualProtectEx使用此句柄修改进程的虚拟内存
        private const uint PROCESS_VM_READ = 0x0010;//允许函数访问权限
        private const uint PROCESS_VM_WRITE = 0x0020;//允许函数写入权限

        private const uint MEM_COMMIT = 0x1000;//为特定的页面区域分配内存中或磁盘的页面文件中的物理存储
        private const uint MEM_RELEASE = 0x8000;
        private const uint MEM_RESERVE = 0x2000;//保留进程的虚拟地址空间,而不分配任何物理存储

        private const uint PAGE_READWRITE = 4;

        private int LVIF_TEXT = 0x0001;
        private int LVIF_STATE = 0x0008;
        

        [DllImport("user32.dll")]//查找窗口
        private static extern int FindWindow(
                                            string strClassName,    //窗口类名
                                            string strWindowName    //窗口标题
        );

        [DllImport("user32.dll")]//在窗口列表中寻找与指定条件相符的第一个子窗口
        private static extern int FindWindowEx(
                                              int hwndParent, // handle to parent window
　　                                          int hwndChildAfter, // handle to child window
                                              string className, //窗口类名            
                                              string windowName // 窗口标题
        );
        [DllImport("user32.DLL")]
        private static extern int SendMessage(int hWnd, uint Msg, int wParam, IntPtr lParam);
        [DllImport("user32.dll")]//找出某个窗口的创建者(线程或进程),返回创建者的标志符
        private static extern int GetWindowThreadProcessId(int hwnd, out int processId);
        [DllImport("kernel32.dll")]//打开一个已存在的进程对象,并返回进程的句柄
        private static extern int OpenProcess(uint dwDesiredAccess, bool bInheritHandle, int processId);
        [DllImport("kernel32.dll")]//为指定的进程分配内存地址:成功则返回分配内存的首地址
        private static extern IntPtr VirtualAllocEx(int hProcess, IntPtr lpAddress, uint dwSize, uint flAllocationType, uint flProtect);
        [DllImport("kernel32.dll")]//从指定内存中读取字节集数据
        private static extern bool ReadProcessMemory(
                                            int hProcess, //被读取者的进程句柄
                                            IntPtr lpBaseAddress,//开始读取的内存地址
                                            IntPtr lpBuffer, //数据存储变量
                                            int nSize, //要写入多少字节
                                            ref uint vNumberOfBytesRead//读取长度
        );
        [DllImport("kernel32.dll")]//将数据写入内存中
        private static extern bool WriteProcessMemory(
                                            int hProcess,//由OpenProcess返回的进程句柄
                                            IntPtr lpBaseAddress, //要写的内存首地址,再写入之前,此函数将先检查目标地址是否可用,并能容纳待写入的数据
                                            IntPtr lpBuffer, //指向要写的数据的指针
                                            int nSize, //要写入的字节数
                                            ref uint vNumberOfBytesRead
        );
        [DllImport("kernel32.dll")]
        private static extern bool CloseHandle(int handle);
        [DllImport("kernel32.dll")]//在其它进程中释放申请的虚拟内存空间
        private static extern bool VirtualFreeEx(
                                    int hProcess,//目标进程的句柄,该句柄必须拥有PROCESS_VM_OPERATION的权限
                                    IntPtr lpAddress,//指向要释放的虚拟内存空间首地址的指针
                                    uint dwSize,
                                    uint dwFreeType//释放类型
        );
        /// <summary>
        /// LVITEM结构体,是列表视图控件的一个重要的数据结构
        /// 占空间：4(int)x7=28个byte
        /// </summary>
        private struct LVITEM  //结构体
        {
            public int mask;//说明此结构中哪些成员是有效的
            public int iItem;//项目的索引值(可以视为行号)从0开始
            public int iSubItem; //子项的索引值(可以视为列号)从0开始
            public int state;//子项的状态
            public int stateMask; //状态有效的屏蔽位
            public IntPtr pszText;  //主项或子项的名称
            public int cchTextMax;//pszText所指向的缓冲区大小
        }

        private struct LVITEM64  //结构体
        {
            public Int32 mask;
            public Int32 iItem;
            public Int32 iSubItem;
            public Int32 state;
            public Int32 stateMask;
            public IntPtr pszText;
            public Int32 cchTextMax;
            public Int32 iImage;
            public Int32 lParam;
            public Int32 iIndent;
        }




        /// <summary>  
        /// LV列表总行数
        /// </summary>
        private int ListView_GetItemRows(int handle)
        {
            return SendMessage(handle, LVM_GETITEMCOUNT, 0, IntPtr.Zero);
        }
        /// <summary>  
        /// LV列表总列数
        /// </summary>
        private int ListView_GetItemCols(int handle)
        {
            return SendMessage(handle, HDM_GETITEMCOUNT, 0, IntPtr.Zero);
        }

        [DllImport("user32.dll", EntryPoint = "SendMessage", CharSet = CharSet.Auto)]
        private static extern int SendMessageLVItem(int hWnd, uint msg, int wParam, IntPtr lvi);

        [DllImport("kernel32.dll", EntryPoint = "WriteProcessMemory", CharSet = CharSet.Auto)]
        private static extern bool WriteProcessMemory2(
                                    int hProcess,//由OpenProcess返回的进程句柄
                                    IntPtr lpBaseAddress, //要写的内存首地址,再写入之前,此函数将先检查目标地址是否可用,并能容纳待写入的数据
                                    ref  ListUtil.LVITEM lvi, //指向要写的数据的指针
                                    int nSize, //要写入的字节数
                                    ref uint vNumberOfBytesRead);

        private const int LVIS_UNCHECKED = 0x1000;
        private const int LVIS_CHECKED = 0x2;
        private const int LVIS_CHECKEDMASK = 0x3;

        /**
         * 
         * 对 SysListView32 进行选择
         * 
         * 发送 LVM_SETITEMSTATE 选择项目, 注意需要提前吧参数写入目标进程
         * 
         * 
         * index : 要选中得索引
         **/
        public int selectItem2(int index)
        { 
        
            // 查找句柄
            hwnd = FindWindow("#32770", "会话");
            //进程界面窗口的句柄,通过SPY获取
            hwnd = FindWindowEx(hwnd, 0, "SysListView32", null);


            // 打开进程并分配内存
            int processId2;
            GetWindowThreadProcessId(hwnd, out processId2);
            int process2 = OpenProcess(PROCESS_VM_OPERATION | PROCESS_VM_READ | PROCESS_VM_WRITE, false, processId2);
            IntPtr pointer2 = VirtualAllocEx(process2, IntPtr.Zero, 4096, MEM_RESERVE | MEM_COMMIT, PAGE_READWRITE);


            LVITEM vItem = new LVITEM();
            vItem.mask = LVIF_STATE;
            vItem.state = LVIS_CHECKEDMASK; 
            vItem.stateMask = LVIS_CHECKEDMASK;

            // 写出配置到目标进程
            uint out2 = 0;
            WriteProcessMemory2(process2, pointer2, ref vItem, Marshal.SizeOf(typeof(LVITEM)), ref out2);

            // 发送消息
            int res = SendMessageLVItem(hwnd, LVM_SETITEMSTATE, 2,  pointer2);//listview的列头句柄

            // 释放内存
            VirtualFreeEx(process2, pointer2, 0, MEM_RELEASE);
            return 99;
            
        }


        /**
         * 获取 SysListView32 组件得内容
         * 
         * 返回二维数组, 行+列
         * 
         **/
        public string[,] getText(String ukeyid)
        {
            int headerhwnd; //listview控件的列头句柄
            int rows, cols;  //listview控件中的行列数
            int processId = -1; //进程pid  


            //进程界面窗口的句柄,通过SPY获取
            hwnd = FindWindow("#32770", "会话");
            hwnd = FindWindowEx(hwnd, 0, "SysListView32", null);
            //listview的列头句柄
            headerhwnd = SendMessage(hwnd, LVM_GETHEADER, 0, IntPtr.Zero);


            rows = ListView_GetItemRows(hwnd);//总行数
            cols = ListView_GetItemCols(headerhwnd);//列表列数
            GetWindowThreadProcessId(hwnd, out processId);

            //打开并插入进程
            process = OpenProcess(PROCESS_VM_OPERATION | PROCESS_VM_READ | PROCESS_VM_WRITE, false, processId);
            //申请代码的内存区,返回申请到的虚拟内存首地址
            string[,] tempStr;//二维数组
            string[] temp = new string[cols];

            //将要读取的其他程序中的ListView控件中的文本内容保存到二维数组中
            tempStr = GetListViewItmeValue(rows, cols);

            /**
            int index = -1;
            String s = "";
            for (int i = 0; i < rows; i++)
            {
                for (int j = 0; j < cols; j++)
                {
                    temp[j] = tempStr[i, j];
                    s += tempStr[i, j] + ",";
                    if (tempStr[i,j].Contains(ukeyid))
                    {
                        index = i;
                        break;
                    }
                }


                s += "\n";
                if(index != -1)
                {
                    break;
                }
            }
            **/
            return tempStr;
        }

        /// <summary>
        /// 从内存中读取指定的LV控件的文本内容
        /// </summary>
        /// <param name="rows">要读取的LV控件的行数</param>
        /// <param name="cols">要读取的LV控件的列数</param>
        /// <returns>取得的LV控件信息</returns>
        private string[,] GetListViewItmeValue(int rows, int cols)
        {
            string[,] tempStr = new string[rows, cols];//二维数组:保存LV控件的文本信息
            for (int i = 0; i < rows; i++)
            {
                for (int j = 0; j < cols; j++)
                {

                    pointer = VirtualAllocEx(process, IntPtr.Zero, 4096, MEM_RESERVE | MEM_COMMIT, PAGE_READWRITE);

                    byte[] vBuffer = new byte[256];//定义一个临时缓冲区
                    LVITEM[] vItem = new LVITEM[1];
                    vItem[0].mask = LVIF_TEXT;//说明pszText是有效的
                    vItem[0].iItem = i;     //行号
                    vItem[0].iSubItem = j;  //列号
                    vItem[0].cchTextMax = vBuffer.Length;//所能存储的最大的文本为256字节
                    vItem[0].pszText = (IntPtr)(pointer + Marshal.SizeOf(typeof(LVITEM)));
                    uint vNumberOfBytesRead = 0;
                    uint vNumberOfBytesRead2 = 0;

                    //把数据写到vItem中
                    //pointer为申请到的内存的首地址
                    //UnsafeAddrOfPinnedArrayElement:获取指定数组中指定索引处的元素的地址
                    WriteProcessMemory(process, pointer, Marshal.UnsafeAddrOfPinnedArrayElement(vItem, 0), Marshal.SizeOf(typeof(LVITEM)), ref vNumberOfBytesRead);

                    //发送LVM_GETITEMW消息给hwnd,将返回的结果写入pointer指向的内存空间
                   int res=  SendMessage(hwnd, LVM_GETITEMW, i, pointer);

                    //从pointer指向的内存地址开始读取数据,写入缓冲区vBuffer中
                    //ReadProcessMemory(process, ((int)pointer + Marshal.SizeOf(typeof(LVITEM))), Marshal.UnsafeAddrOfPinnedArrayElement(vBuffer, 0), vBuffer.Length, ref vNumberOfBytesRead);

                    ReadProcessMemory(process,
                        (IntPtr)(pointer + System.Runtime.InteropServices.Marshal.SizeOf(typeof(LVITEM))),
                      System.Runtime.InteropServices.Marshal.UnsafeAddrOfPinnedArrayElement(vBuffer, 0),
                      vBuffer.Length, ref vNumberOfBytesRead2);


                    string vText = Encoding.Unicode.GetString(vBuffer, 0, (int)vNumberOfBytesRead); ;
                    tempStr[i, j] = vText.Replace("\0", "");
                    if(pointer != IntPtr.Zero)
                    {
                        VirtualFreeEx(process, pointer, 0, MEM_RELEASE);//在其它进程中释放申请的虚拟内存空间,MEM_RELEASE方式很彻底,完全回收

                    }

                }
            }
            CloseHandle(process);//关闭打开的进程对象
            return tempStr;
        }
}
}

```

[参考资料1](https://stackoverflow.com/questions/4271291/writeprocessmemory-with-an-int-value)

[参考资料2](https://stackoverflow.com/questions/5369155/getting-text-from-syslistview32-in-64bit)

[参考资料3](https://www.cnblogs.com/hongfei/archive/2012/12/24/2829799.html)
