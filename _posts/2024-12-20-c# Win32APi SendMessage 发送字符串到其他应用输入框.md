---
layout: post
title:  "c# Win32APi SendMessage 发送字符串到其他应用输入框"
date:   2024-12-20 10:29:20 +0800
categories:
      - c#
      - win32api
tags:
      - SendMessage
      - SendTextMessage
---

主要是留作记录，今天突然有操作windows窗体程序的需求。 需要自动化输入和点击，方便应用操作。

这里记录了如何查找并发送文本到其他应用输入框, 这里重点查看onclick事件中的逻辑。

首先找到应用窗口，然后遍历并保存所有子窗口。最后调用函数发送即可。 实测密码框也可以发送成功。


```c#
using System.Diagnostics;
using System.Runtime.InteropServices;
using System.Text;
using static System.Net.Mime.MediaTypeNames;

namespace testdemo
{
    public partial class Form1 : Form
    {
        public Form1()
        {
            InitializeComponent();
        }
        private List<IntPtr> hwnds = new List<IntPtr>();
 
        private void button1_Click(object sender, EventArgs e)
        {

            const uint WM_KEYDOWN = 0x100;

            int hwnd1 = 0;
            int hwndbtn = 0;
            int hwndtxt = 0;
            int hwnm1 = 0;
            int hwnm2 = 0;
            // 查找窗口
            hwnd1 = FindWindow("WindowsForms10.Window.8.app.0.19fd5c7_r3_ad1", "Form1aaaee");
            //hwnd1 = FindWindow("WindowsForms10.Window.8.app.0.1e09f85_r7_ad1", "AreaCity Geo格式转换工具 Ver:1.3.240505");
            Console.WriteLine("============");

            //遍历所有子窗口并保存
            EnumWindowProc childProc = new EnumWindowProc(enumWindowProca);
            EnumChildWindows(hwnd1, childProc, IntPtr.Zero);
            
            // 发送字符串到指定输入框消息
            SendTextMessage(hwnds[0], WM_SETTEXT, 0, UnicodeEncoding.Unicode.GetBytes("ha啊我"));



        }
        private  bool enumWindowProca(IntPtr hwnd, IntPtr lParam) {
            StringBuilder s = new StringBuilder();
            GetClassName(hwnd, s, 100);
            Console.WriteLine(s.ToString()) ;
            Debug.WriteLine(s.ToString());
            if (s.ToString().Equals("WindowsForms10.Edit.app.0.19fd5c7_r3_ad1")) {
                hwnds.Add(hwnd);
            }

            return true;
           }
        [DllImport("user32")]
        private static extern int GetWindowText(IntPtr hWnd, StringBuilder lptrString, int nMaxCount);
        [DllImport("user32")]
        private static extern int GetClassName(IntPtr hWnd, StringBuilder lptrString, int nMaxCount);

        private delegate bool EnumWindowProc(IntPtr hwnd, IntPtr lParam);

        [DllImport("user32")]
        [return: MarshalAs(UnmanagedType.Bool)]
        private static extern bool EnumChildWindows(int window, EnumWindowProc callback, IntPtr lParam);



        [DllImport("user32.dll", EntryPoint = "SendMessage", CharSet = CharSet.Auto)]
        private static extern int SendTextMessage(IntPtr hWnd, uint Msg, int wParam, byte[] lParam);


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

        [DllImport("user32.DLL", SetLastError = true, CharSet = CharSet.Unicode)]
        private static extern int SendMessage(int hWnd, uint Msg, int wParam, IntPtr lParam);

        [DllImport("user32.dll", EntryPoint = "SendMessage", CharSet = CharSet.Auto)]
        private static extern int SendMessageFindItem(int hWnd, uint Msg, int wParam, ref tagLVFINDINFOA lParam);

        [DllImport("user32.DLL")]
        private static extern int SendMessageKey(int hWnd, uint Msg, IntPtr wParam, IntPtr lParam);

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
    }


}
```
