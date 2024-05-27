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
using Newtonsoft.Json.Linq;
using System;
using System.Collections.Generic;
using System.Diagnostics;
using System.Drawing;
using System.Linq;
using System.Reflection;
using System.Reflection.Emit;
using System.Runtime.InteropServices;
using System.Text;
using System.Threading.Tasks;
using static System.Net.Mime.MediaTypeNames;

namespace spiderClient
{
    internal class ListUtil
    {

        // int hwnd;   //窗口句柄
        // int process;//进程句柄
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


        [DllImport("user32.dll")]//在窗口列表中寻找与指定条件相符的第一个子窗口
        private static extern int FindWindowExW(
                                              int hwndParent, // handle to parent window
　　                                          int hwndChildAfter, // handle to child window
                                              string className, //窗口类名            
                                              string windowName // 窗口标题
        );
        [DllImport("user32.DLL", EntryPoint = "SendMessage", SetLastError = true)]
        private static extern int SendMessage0(int hWnd, uint Msg, int wParam, StringBuilder lParam);

        [DllImport("user32.DLL", SetLastError =true, CharSet =CharSet.Unicode)]
        private static extern int SendMessage(int hWnd, uint Msg, int wParam, IntPtr lParam);

        [DllImport("user32.dll", EntryPoint = "SendMessage", CharSet = CharSet.Auto)]
        private static extern int SendMessageFindItem(int hWnd, uint Msg, int wParam, ref tagLVFINDINFOA lParam);

        [DllImport("user32.DLL")]
        private static extern int SendMessageKey(int hWnd, uint Msg, IntPtr wParam, IntPtr lParam);


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
            public int pszText;  //主项或子项的名称
            public int cchTextMax;//pszText所指向的缓冲区大小
        }

       


        public struct COPYDATASTRUCT
        {
            public IntPtr dwData; //可以是任意值
            public int cbData;    //指定lpData内存区域的字节数
            [MarshalAs(UnmanagedType.LPStr)]
            public string lpData; //发送给目录窗口所在进程的数据
        }

        const uint WM_SETTEXT = 0x000C;

        const int BM_CLICK = 0xF5;  



        public bool login(String ukeyid, string pwd, string executeid)
        {
            bool logined = false;

            List<UKey>  s = Utils.getConfig();
            if(s == null || s.Count == 0)
            {
                Program.log.Information(String.Format("收到登陆请求 {0}, 但未配置加密狗信息", ukeyid));
                return false;
            }

            for (int i3 = 0; i3 < 30 && logined == false; i3++)
            {
                Thread.Sleep(3000);
                int hwnd1 = FindWindow("#32770", "选择证书");
                int hwnd2 = FindWindow("#32770", "请输入Usbkey访问密码");

                if(hwnd1 == 0 && hwnd2 == 0)
                {
                    Program.log.Information(String.Format("[{0}] 第{1}次,未识别到登录窗口", ukeyid, i3));

                    continue;
                } else
                {

                    if(hwnd1 != 0)
                    {
                        Program.log.Information(String.Format("[{0}] 第{1}次,识别到多个加密狗,登录窗口", ukeyid, i3));
                        string[,] strings = getText(ukeyid);
                        int index = -1;



                        int rows = strings.GetLength(0);
                        int columns = strings.GetLength(1);

                        string msgs = "";
                        bool hasukey = false;
                        for (int i = 0; i < strings.GetLength(0); i++)
                        {
                            for (int j = 0; j < strings.GetLength(1); j++)
                            {

                                string tmp = strings[i, j];
                                msgs += tmp + ",";
                                if (tmp.Contains(ukeyid))
                                {
                                    hasukey = true;
                                    selectItem2(i);
                                    btnclick(1);

                                    for (int i1 = 0; i1 < 30 && logined == false; i1++)
                                    {
                                        Thread.Sleep(1000);

                                        // 查找密码窗口
                                        int hwnd0 = FindWindow("#32770", "请输入Usbkey访问密码");
                                        if (hwnd0 == 0)
                                        {
                                            Program.log.Information(String.Format("[{0}] 第{1}次,未识别到密码窗口", ukeyid, i1));

                                            continue;
                                        }
                                        else
                                        {
                                            Program.log.Information(String.Format("[{0}] 第{1}次,识别到密码窗口, 发送登录操作", ukeyid, i1));
                                            bool res = inputpasswordAndLogin(ukeyid, pwd, hwnd0, executeid);
                                            if (res)
                                            {
                                                logined = true;
                                                Utils.sendmsg(3, ukeyid, executeid);
                                                
                                            } 
                                        }
                                    }

                                }

                            }
                            msgs += "\n";
                        }
                        Program.log.Information(String.Format("[{0}] 加密狗读取信息 {1}", ukeyid, msgs));

                        if (hasukey == false)
                        {
                            Program.log.Information(String.Format("[{0}] 登录时,未检测到加密狗存在", ukeyid));
                            Utils.sendmsg(4, ukeyid, String.Format("[{0}] 登录时,未检测到加密狗存在", ukeyid), executeid);
                            btnclick(5);

                        }
                    }

                    if(hwnd2 != 0)
                    {
                        Program.log.Information(String.Format("[{0}] 登录时,直接检测到密码窗口,只有一个加密狗", ukeyid));


                        int hwnd4 = 0;
                        do
                        {
                            hwnd4 = FindWindowEx(hwnd2, hwnd4, "Edit", "");
                            if (hwnd4 != 0)
                            {

                                int titleSize = SendMessage(hwnd4, WM_GETTEXTLENGTH, 0, IntPtr.Zero);
                                StringBuilder title = new StringBuilder(titleSize + 1);
                                SendMessage0(hwnd4, (int)WM_GETTEXT, title.Capacity, title);
                                if(title.ToString().Contains(ukeyid))
                                {
                                    bool res = inputpasswordAndLogin(ukeyid, pwd, hwnd2, executeid);
                                    if (res)
                                    {
                                        logined = true;
                                        Utils.sendmsg(3,executeid);
                                    }
                                }
                            }
                            else
                            {
                                break;

                            }

                        } while (hwnd4 != 0);
                        Utils.sendmsg(4, ukeyid, 
                            String.Format("[{0}] 登录时,只有一个加密狗, 不是当前请求登录的用户，匹配失败", executeid),
                            executeid);
                        Program.log.Information(String.Format("[{0}] 登录时,只有一个加密狗, 不是当前请求登录的用户，匹配失败", ukeyid));


                      

                    }
                }


            }


            if(logined == false)
            {
                Utils.sendmsg(4, ukeyid, String.Format("[{0}] 90s内未检测到登录窗口弹出", ukeyid), executeid);
                Program.log.Information(String.Format("[{0}] 90s内未检测到登录窗口弹出", ukeyid));
            }
            
            return true;

        }
        [DllImport("user32.dll")]
        private static extern int GetDlgCtrlID(IntPtr hwnd);
      

        [DllImport("user32.dll", CharSet = CharSet.Unicode, SetLastError = true)]
        static extern int GetWindowText(IntPtr hWnd, StringBuilder lpString,int nMaxCount);

        [DllImport("user32", SetLastError = true, CharSet = CharSet.Auto)]
        private extern static int GetWindowTextLength(IntPtr hWnd);
        [DllImport("user32", SetLastError = true, CharSet = CharSet.Auto)]
        private extern static int GetWindowTextLengthW(IntPtr hWnd);
        
        const uint WM_GETTEXT = 0x000D;
        const uint WM_GETTEXTLENGTH = 0x000E;


        /**
         * 
         * 获取edit 控件内容
         * */
        
        public int getEditText()
        {

            /*        int hwnd2 = FindWindow("#32770", "请输入Usbkey访问密码");
                    int hwnd0 = FindWindowEx(hwnd2, 0, "Edit", "27122d1829770979");*/


            int hwnd2 = FindWindow("#32770", "新建会话属性");
            int hwnd3 = FindWindowEx(hwnd2, 0, "#32770", "");


            // 打开进程并分配内存
            // int processId2;
            //  GetWindowThreadProcessId(hwnd3, out processId2);
            // int process2 = OpenProcess(PROCESS_VM_OPERATION | PROCESS_VM_READ | PROCESS_VM_WRITE, false, processId2);
            // IntPtr txtpointer = VirtualAllocEx(process2, IntPtr.Zero, 4096, MEM_RESERVE | MEM_COMMIT, PAGE_READWRITE);
            StringBuilder s = new StringBuilder();

            /*            GetWindowText((IntPtr)0x000404, s, 126);
                        MessageBox.Show(s.ToString());*/

            int hwnd4 = 0;
            do
            {
                hwnd4 = FindWindowEx(hwnd3, hwnd4, "Edit", "");
                if (hwnd4 != 0)
                {
           
                    int titleSize = SendMessage(hwnd4, WM_GETTEXTLENGTH, 0, IntPtr.Zero);
                    StringBuilder title = new StringBuilder(titleSize + 100);
                    int res22= SendMessage0(hwnd4, (int)WM_GETTEXT, title.Capacity, title);
                    uint out2 = 0;

                    MessageBox.Show(title.ToString() + "_" + titleSize + "_res22," + res22);

                }
                else
                {
                    break;

                }


            } while (hwnd2 != 0);
            //VirtualFreeEx(process2, txtpointer, 0, MEM_RELEASE);

            return 0;
        }

        private bool inputpasswordAndLogin(String ukeyid, string pwd, int hwnd0, string executeid)
        {
            SendKeystroke(pwd);
            Thread.Sleep(1000);
            hwnd0 = FindWindowEx(hwnd0, 0, "Button", "确定");
            if (hwnd0 != 0)
            {
                btnclick(2);
                Program.log.Information(String.Format("[{0}]  登录成功", ukeyid));
                
                return true;
            }
            else
            {
                Utils.sendmsg(4, ukeyid, String.Format("[{0}] 未识别到密码窗口确认按钮", ukeyid), executeid);


                Program.log.Information(String.Format("[{0}]  未识别到密码窗口确认按钮", ukeyid));
                return false;
            }
        }

        /**
         * 按钮点击事件
         * */
        public void btnclick(int type)
        {
            int hwnd1 = 0;

            if (type == 1)
            {
                // 查找句柄
                hwnd1 = FindWindow("#32770", "选择证书");
                if(hwnd1 != 0)
                {
                    hwnd1 = FindWindowEx(hwnd1, 0, "Button", "确定");

                    SendMessage(hwnd1, BM_CLICK, 0, IntPtr.Zero);
                }
           

            }

            if(type == 2) {
                // 密码登录
                hwnd1 = FindWindow("#32770", "请输入Usbkey访问密码");
                if(hwnd1 != 0)
                {
                    hwnd1 = FindWindowEx(hwnd1, 0, "Button", "确定");

                    SendMessage(hwnd1, BM_CLICK, 0, IntPtr.Zero);
                }
           
            }
            if (type == 5)
            {
                   // 查找句柄
                hwnd1 = FindWindow("#32770", "选择证书");
                if(hwnd1 != 0)
                {
                    hwnd1 = FindWindowEx(hwnd1, 0, "Button", "取消");

                    SendMessage(hwnd1, BM_CLICK, 0, IntPtr.Zero);
                }

            }
        }
        [DllImport("user32.dll")]
        static extern bool PostMessage(int hWnd, uint Msg, int wParam, int lParam);


        /**
         * 
         * 发送按键消息
         * 
         * 不过只能发送ascii码
         * */
        public void SendKeystroke(String txt)
        {
            const uint WM_KEYDOWN = 0x100;

            int hwnd1 = 0;
            int hwnd2 = 0;
            // 查找句柄
            hwnd1 = FindWindow("#32770", "请输入Usbkey访问密码");
           
            if (hwnd1 != 0)
            {
                for (int i = 0; i < 4; i++)
                {
                    hwnd2 = FindWindowEx(hwnd1, hwnd2, "Edit", "");
                }
                byte[] asciiBytes = Encoding.ASCII.GetBytes(txt);
                for (int i1 = 0; i1 < asciiBytes.Length; i1++)
                {
                    PostMessage(hwnd2, WM_KEYDOWN, (int)asciiBytes[i1], 0);
                }
            }

        }


        /**
         * 
         * 发送按键消息
         * 
         * 
         * 首先定位标签，然后根据结构查询编辑框
         * 不过只能发送ascii码
         * */
        public void SendKeystroke2(String txt)
        {
            const uint WM_KEYDOWN = 0x100;

            int hwnd1 = 0;
            int hwnd2 = 0;
            // 查找句柄
            //hwnd1 = FindWindow("#32770", "请输入Usbkey访问密码");
            hwnd1 = FindWindow("#32770", "新建会话属性");
            hwnd1 = FindWindowEx(hwnd1, 0 , "#32770", "");
  
            //hwnd1 = FindWindowEx(hwnd1,0, "#32770", "");
            hwnd2 = FindWindowEx(hwnd1, 0, "Static", "主机(&H):");
            hwnd2 = FindWindowEx(hwnd1, hwnd2, "Edit", "");
            byte[] asciiBytes = Encoding.ASCII.GetBytes(txt);
            for (int i1 = 0; i1 < asciiBytes.Length; i1++)
            {
                PostMessage(hwnd2, WM_KEYDOWN, (int)asciiBytes[i1], 0);
            }
           

        }

        private struct tagPoint   //结构体
        {
            public uint x;
            public uint y;

        }


        /// <summary>
        /// LVITEM结构体,是列表视图控件的一个重要的数据结构
        /// 占空间：4(int)x7=28个byte
        /// </summary>
        private struct tagLVFINDINFOA   //结构体
        {
            public uint flag;
            public IntPtr psz;
         
        }


        uint LVFI_PARAM  = 0x1;
        uint LVFI_STRING = 0x0002; // 完全匹配
        uint LVFI_SUBSTRING = 0x0004;  // Same as LVFI_PARTIAL
        uint LVFI_PARTIAL = 0x0008;  // 以psz字符从开始匹配
        uint LVM_FINDITEMA = (LVM_FIRST + 13);
        uint LVM_FINDITEMW = (LVM_FIRST + 83);
        [DllImport("kernel32.dll ")]
        static extern bool ReadProcessMemory(int hProcess, int lpBaseAddress, out int lpBuffer, int nSize, out int lpNumberOfBytesRead);


        /**
         * 
         * 在 sys32tree 上查询指定字符串，
         * 返回查询结果行数
         * 
         * 有点奇怪的是，这个函数不知道为啥，只能查找第一列的数据。搞不懂
         * */
        public void findTextOnSys32tree(String txt)
        {

            // 查找句柄
            //int hwnd = FindWindow("#32770", "选择证书");

              int hwnd = FindWindow("#32770", "会话");

            //进程界面窗口的句柄,通过SPY获取
            hwnd = FindWindowEx(hwnd, 0, "SysListView32", null);



            // 打开进程并分配内存
            int processId2;
            GetWindowThreadProcessId(hwnd, out processId2);
            int process2 = OpenProcess(PROCESS_VM_OPERATION | PROCESS_VM_READ | PROCESS_VM_WRITE, false, processId2);
            IntPtr txtpointer = VirtualAllocEx(process2, IntPtr.Zero, 4096, MEM_RESERVE | MEM_COMMIT, PAGE_READWRITE);
            IntPtr strctpointer = VirtualAllocEx(process2, IntPtr.Zero, 4096, MEM_RESERVE | MEM_COMMIT, PAGE_READWRITE);


            String s = txt ;


            byte[] s2 = UnicodeEncoding.Unicode.GetBytes(s);
            // 写出配置到目标进程
            uint out2 = 0;
            bool res = false;
            res =WriteProcessMemory3(process2, txtpointer,  s2, s2.Length, ref out2);

            byte[] buffer22 = new byte[10];
            ReadProcessMemory(process2, txtpointer, Marshal.UnsafeAddrOfPinnedArrayElement(buffer22, 0),10, ref out2);

            tagLVFINDINFOA t = new tagLVFINDINFOA();
            t.flag = LVFI_SUBSTRING;
            t.psz = txtpointer;

            res = WriteProcessMemory4(process2, strctpointer, ref t, Marshal.SizeOf(typeof(tagLVFINDINFOA)), ref out2);


            // 发送消息
            int i = SendMessage(hwnd, LVM_FINDITEMW, -1, strctpointer);
            int a = Marshal.GetLastWin32Error();

            // 释放内存
            VirtualFreeEx(process2, txtpointer, 0, MEM_RELEASE);
            VirtualFreeEx(process2, strctpointer, 0, MEM_RELEASE);
            MessageBox.Show("res " + i);
            int c = i;

        }

        public void sendText(String text)
        {
            int hwnd1 = 0;
            int hwnd2 = 0;
            // 查找句柄
            hwnd1 = FindWindow("#32770", "请输入Usbkey访问密码");
            if(hwnd1 != 0)
            {
                for (int i = 0; i < 4; i++)
                {
                    hwnd2 = FindWindowEx(hwnd1, hwnd2, "Edit", "");
                }
                SendTextMessage(hwnd2, WM_SETTEXT, 0, UnicodeEncoding.Unicode.GetBytes(text));
            }
        

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



        [DllImport("user32.dll", EntryPoint = "SendMessage", CharSet = CharSet.Auto)]
        private static extern int SendTextMessage(int hWnd, uint Msg, int wParam,  byte[] lParam);


        [DllImport("kernel32.dll", EntryPoint = "WriteProcessMemory", CharSet = CharSet.Auto)]
        private static extern bool WriteProcessMemory2(
                                    int hProcess,//由OpenProcess返回的进程句柄
                                    IntPtr lpBaseAddress, //要写的内存首地址,再写入之前,此函数将先检查目标地址是否可用,并能容纳待写入的数据
                                    ref  ListUtil.LVITEM lvi, //指向要写的数据的指针
                                    int nSize, //要写入的字节数
                                    ref uint vNumberOfBytesRead);
        [DllImport("kernel32.dll", EntryPoint = "WriteProcessMemory", CharSet = CharSet.Auto)]
        private static extern bool WriteProcessMemory3(
                            int hProcess,//由OpenProcess返回的进程句柄
                            IntPtr lpBaseAddress, //要写的内存首地址,再写入之前,此函数将先检查目标地址是否可用,并能容纳待写入的数据
                            byte[] lvi, //指向要写的数据的指针
                            int nSize, //要写入的字节数
                            ref uint vNumberOfBytesRead);

        [DllImport("kernel32.dll", EntryPoint = "WriteProcessMemory", CharSet = CharSet.Auto)]
        private static extern bool WriteProcessMemory4(
                        int hProcess,//由OpenProcess返回的进程句柄
                        IntPtr lpBaseAddress, //要写的内存首地址,再写入之前,此函数将先检查目标地址是否可用,并能容纳待写入的数据
                        ref tagLVFINDINFOA lvi, //指向要写的数据的指针
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
            int hwnd = FindWindow("#32770", "选择证书");

            //  int hwnd = FindWindow("#32770", "会话");

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
            int res = SendMessageLVItem(hwnd, LVM_SETITEMSTATE, index,  pointer2);//listview的列头句柄

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
            int hwnd = FindWindow("#32770", "选择证书");
            Program.log.Information(String.Format("选择证书句柄: {0}", hwnd));
            hwnd = FindWindowEx(hwnd, 0, "SysListView32", null);
            Program.log.Information(String.Format("SysListView32句柄: {0}", hwnd));
            //listview的列头句柄
            headerhwnd = SendMessage(hwnd, LVM_GETHEADER, 0, IntPtr.Zero);


            rows = ListView_GetItemRows(hwnd);//总行数
            cols = ListView_GetItemCols(headerhwnd);//列表列数
            Program.log.Information(String.Format("SysListView32 rows: {0}, cols: {1}", rows, cols));
            GetWindowThreadProcessId(hwnd, out processId);
            //打开并插入进程
            int process = OpenProcess(PROCESS_VM_OPERATION | PROCESS_VM_READ | PROCESS_VM_WRITE, false, processId);

            Program.log.Information(String.Format("GetWindowThreadProcessId : {0}, openHandle:{1}", processId, process));

            //申请代码的内存区,返回申请到的虚拟内存首地址
            string[,] tempStr;//二维数组
            string[] temp = new string[cols];

            //将要读取的其他程序中的ListView控件中的文本内容保存到二维数组中
            tempStr = GetListViewItmeValue(rows, cols, process, hwnd);

        
            return tempStr;
        }

        

    public string[] getTegetColumxt2(int col)
        {
            int headerhwnd; //listview控件的列头句柄
            int rows, cols;  //listview控件中的行列数
            int processId = -1; //进程pid  


            //进程界面窗口的句柄,通过SPY获取
            int hwnd = FindWindow("#32770", "会话");
            Program.log.Information(String.Format("选择证书句柄: {0}", hwnd));
            hwnd = FindWindowEx(hwnd, 0, "SysListView32", null);
            Program.log.Information(String.Format("SysListView32句柄: {0}", hwnd));
            //listview的列头句柄
            headerhwnd = SendMessage(hwnd, LVM_GETHEADER, 0, IntPtr.Zero);


            rows = ListView_GetItemRows(hwnd);//总行数
            cols = ListView_GetItemCols(headerhwnd);//列表列数
            Program.log.Information(String.Format("SysListView32 rows: {0}, cols: {1}", rows, cols));
            GetWindowThreadProcessId(hwnd, out processId);
            //打开并插入进程
            int process = OpenProcess(PROCESS_VM_OPERATION | PROCESS_VM_READ | PROCESS_VM_WRITE, false, processId);

            Program.log.Information(String.Format("GetWindowThreadProcessId : {0}, openHandle:{1}", processId, process));


            //将要读取的其他程序中的ListView控件中的文本内容保存到二维数组中
            return getColumValue(rows, cols, process, hwnd, col);


        }

         struct tagLVCOLUMNA
        {
            public uint mask;
            public int fmt;
            public int cx;
            public uint pszText;
            public uint cchText;
            public uint iSubItem;


        }

         uint LVCF_TEXT  = 0x4;
        uint LVCF_SUBITEM = 0x8;
        uint LVM_GETCOLUMNA = (LVM_FIRST + 25);
        uint LVM_GETCOLUMNW = (LVM_FIRST + 95);

        /// <summary>
        /// 从内存中读取指定的LV控件的文本内容
        /// </summary>
        /// <param name="rows">要读取的LV控件的行数</param>
        /// <param name="cols">要读取的LV控件的列数</param>
        /// <returns>取得的LV控件信息</returns>
        private string[] getColumValue(int rows, int cols, int process, int hwnd, int col)
        {
            IntPtr pointer;

            string[,] tempStr = new string[rows, cols];//二维数组:保存LV控件的文本信息
            string[] tempStr2 = new string[rows];//二维数组:保存LV控件的文本信息

            pointer = VirtualAllocEx(process, IntPtr.Zero, 4096, MEM_RESERVE | MEM_COMMIT, PAGE_READWRITE);

            byte[] vBuffer = new byte[512];//定义一个临时缓冲区
            tagLVCOLUMNA[] vItem = new tagLVCOLUMNA[1];
            vItem[0].mask = LVCF_TEXT | LVCF_SUBITEM;
            vItem[0].pszText = (UInt32)(pointer + Marshal.SizeOf(typeof(tagLVCOLUMNA)));
            vItem[0].cchText = 512;
            vItem[0].iSubItem = 2;  //列号

            uint vNumberOfBytesRead = 0;
            uint vNumberOfBytesRead2 = 0;

            //把数据写到vItem中
            //pointer为申请到的内存的首地址
            //UnsafeAddrOfPinnedArrayElement:获取指定数组中指定索引处的元素的地址
            WriteProcessMemory(process, pointer, Marshal.UnsafeAddrOfPinnedArrayElement(vItem, 0), Marshal.SizeOf(typeof(tagLVCOLUMNA)), ref vNumberOfBytesRead);

            //发送LVM_GETITEMW消息给hwnd,将返回的结果写入pointer指向的内存空间
            int res = SendMessage(hwnd, LVM_GETCOLUMNW, col, pointer);



            ReadProcessMemory(process,
                (IntPtr)(pointer + System.Runtime.InteropServices.Marshal.SizeOf(typeof(tagLVCOLUMNA))),
                System.Runtime.InteropServices.Marshal.UnsafeAddrOfPinnedArrayElement(vBuffer, 0),
                vBuffer.Length, ref vNumberOfBytesRead);


            string vText = Encoding.Unicode.GetString(vBuffer, 0, (int)vNumberOfBytesRead); ;
            if (pointer != IntPtr.Zero)
            {
                VirtualFreeEx(process, pointer, 0, MEM_RELEASE);//在其它进程中释放申请的虚拟内存空间,MEM_RELEASE方式很彻底,完全回收

            }

            CloseHandle(process);//关闭打开的进程对象
            return tempStr2;
        }

        public string[,] getText2(String ukeyid)
        {
            int headerhwnd; //listview控件的列头句柄
            int rows, cols;  //listview控件中的行列数
            int processId = -1; //进程pid  


            //进程界面窗口的句柄,通过SPY获取
            int hwnd = FindWindow("#32770", "会话");
            Program.log.Information(String.Format("选择证书句柄: {0}", hwnd));
            hwnd = FindWindowEx(hwnd, 0, "SysListView32", null);
            Program.log.Information(String.Format("SysListView32句柄: {0}", hwnd));
            //listview的列头句柄
            headerhwnd = SendMessage(hwnd, LVM_GETHEADER, 0, IntPtr.Zero);


            rows = ListView_GetItemRows(hwnd);//总行数
            cols = ListView_GetItemCols(headerhwnd);//列表列数
            Program.log.Information(String.Format("SysListView32 rows: {0}, cols: {1}", rows, cols));
            GetWindowThreadProcessId(hwnd, out processId);
            //打开并插入进程
            int process = OpenProcess(PROCESS_VM_OPERATION | PROCESS_VM_READ | PROCESS_VM_WRITE, false, processId);

            Program.log.Information(String.Format("GetWindowThreadProcessId : {0}, openHandle:{1}", processId, process));

            //申请代码的内存区,返回申请到的虚拟内存首地址
            string[,] tempStr;//二维数组
            string[] temp = new string[cols];

            //将要读取的其他程序中的ListView控件中的文本内容保存到二维数组中
            tempStr = GetListViewItmeValue(rows, cols, process, hwnd);


            return tempStr;
        }

        /// <summary>
        /// 从内存中读取指定的LV控件的文本内容
        /// </summary>
        /// <param name="rows">要读取的LV控件的行数</param>
        /// <param name="cols">要读取的LV控件的列数</param>
        /// <returns>取得的LV控件信息</returns>
        private string[,] GetListViewItmeValue(int rows, int cols, int process, int hwnd)
        {
            IntPtr pointer;

            string[,] tempStr = new string[rows, cols];//二维数组:保存LV控件的文本信息
            for (int i = 0; i < rows; i++)
            {
                for (int j = 0; j < cols; j++)
                {

                    pointer = VirtualAllocEx(process, IntPtr.Zero, 4096, MEM_RESERVE | MEM_COMMIT, PAGE_READWRITE);

                    byte[] vBuffer = new byte[512];//定义一个临时缓冲区
                    LVITEM[] vItem = new LVITEM[1];
                    vItem[0].mask = LVIF_TEXT;//说明pszText是有效的
                    vItem[0].iItem = i;     //行号
                    vItem[0].iSubItem = j;  //列号
                    vItem[0].cchTextMax = vBuffer.Length;//所能存储的最大的文本为256字节
                    vItem[0].pszText = (Int32)(pointer + Marshal.SizeOf(typeof(LVITEM)));
                    uint vNumberOfBytesRead = 0;
                    uint vNumberOfBytesRead2 = 0;

                    //把数据写到vItem中
                    //pointer为申请到的内存的首地址
                    //UnsafeAddrOfPinnedArrayElement:获取指定数组中指定索引处的元素的地址
                    WriteProcessMemory(process, pointer, Marshal.UnsafeAddrOfPinnedArrayElement(vItem, 0), Marshal.SizeOf(typeof(LVITEM)), ref vNumberOfBytesRead);

                    //发送LVM_GETITEMW消息给hwnd,将返回的结果写入pointer指向的内存空间
                   int res=  SendMessage(hwnd, LVM_GETITEMW, i, pointer);



                    ReadProcessMemory(process,
                        (IntPtr)(pointer + System.Runtime.InteropServices.Marshal.SizeOf(typeof(LVITEM))),
                      System.Runtime.InteropServices.Marshal.UnsafeAddrOfPinnedArrayElement(vBuffer, 0),
                      vBuffer.Length, ref vNumberOfBytesRead);


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
