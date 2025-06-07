# Getting Started

### Existing project (when cex.c exists in the project root directory)
```
1. > cd project_dir
2. > gcc/clang ./cex.c -o ./cex     (need only once, then cex will rebuild itself)
3. > ./cex --help                   get info about available commands
```

### New project / bare cex.h file

1. download [cex.h](https://raw.githubusercontent.com/alexveden/cex/refs/heads/master/cex.h)
2. Make a project directory 
```
mkdir project_dir
cd project_dir
```
3. Make a seed program (NOTE: passing header file is ok)
```
gcc -D CEX_NEW -x c ./cex.h
clang -D CEX_NEW -x c ./cex.h
```
4. Run cex program for project initilization
```
./cex
```
5. Now your project is ready to go 
```
./cex test run all
./cex app run myapp
```
