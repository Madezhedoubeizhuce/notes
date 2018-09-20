# VS Compiling a simple Eigen program

标签（空格分隔）： 软件配置

---

  1. First you need to download Eigen. Using the TortoiseHg Workbench,
    select Clone Repository in the File menu, and use the URL
    https://bitbucket.org/eigen/eigen/ as source. Alternatively,
    download and unpack the zip file. I will denote by EIGENDIR the
    folder in which you store the Eigen library; this folder contains
    other folders like bench, blas, cmake and Eigen.
  2. In Visual Studio, make a new project (File | New | Project) using
    the Empty Project template.
  3. Add a C++ file to the project (Project | Add New Item, choose the
    C++ File template).
  4. Write your program in the file. For instance, copy and paste the
    first program at the [Getting Started guid][1]e. To prevent the console
    window from closing immediately after the program finishes, add
    something like "std::cin.get();" at the end.
  5. Open the project properties page (Project | Properties), in the
    C/C++ folder open the General page, and after Additional Include
    Directories put EIGENDIR (the folder in which you store the Eigen
    library).
  6. Compile the program (Debug | Build Solution, or press F7).
  7. Run the program (Debug | Start debugging, or press F5).


[1]: http://eigen.tuxfamily.org/dox/GettingStarted.html