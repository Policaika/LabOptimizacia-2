```C#
using System.Globalization;
using System.Reflection.Metadata.Ecma335;
using System.Runtime.InteropServices.Marshalling;
using System.Runtime.Intrinsics.Arm;
using System.Threading.Channels;

namespace SimplexMethodBMSTU
{
    class SimplexMethod
    {
        int n; // свободные переменные (столбики)
        int m; // базисные переменные ( cтроки)
        //bool
        // 1. Столбец S
        // 2. Массив с кэфами
        // 3. F строка(целевая функция)
        decimal[] cols = null!; // cтолбец s
        decimal[] cols_new = null!;
        decimal[,] matrix = null!; // массив с кэфами
        decimal[] F = null!;
        string Func = null!; // Определение функции (max/min)
        string TypeLP = null!; // Определение ЛП(ДЗЛП/ПЛП)
        decimal[] F_new = null!;
        string[] basis_variables = null!;
        string[] free_variables = null!;
        string[] basis_variables_dual = null!;
        string[] free_variables_dual = null!;
        decimal[,] matrix_dual_simplex = null!;
        decimal[,] matrix_dual_simplex_new = null!;
        decimal[] cols_dual_simplex = null!;
        decimal[] F_dual_simplex = null!;
        decimal[] F_dual_simplex_new = null!;
        decimal[] cols_dual_simplex_new = null!;
        decimal[,] matrix_new = null!; // массив с кэфами после преобразования
        public SimplexMethod(int n, int m)
        {
            this.n = n;
            this.m = m;
            Make();
        }
        public void Make()
        {
            cols = new decimal[m];
            matrix = new decimal[m, n];
            F = new decimal[n + 1];
            matrix_new = new decimal[m, n];
            F_new = new decimal[n + 1];
            cols_new = new decimal[m];
            basis_variables = new string[m];
            free_variables = new string[n];
            basis_variables_dual = new string[n+1];
            free_variables_dual = new string[m];
            matrix_dual_simplex = new decimal[n,m];
            cols_dual_simplex = new decimal[n+1];
            F_dual_simplex = new decimal[m];
            matrix_dual_simplex_new = new decimal[n,m];
            F_dual_simplex_new = new decimal[m];
            cols_dual_simplex_new = new decimal[n+1];
        }
        public void Input()
        {
            Console.Write("Выберите задачу ЛП(ДЗ/П): ");
            TypeLP = Console.ReadLine();
            Console.Write("Выберите к чему стремится функция (max/min): ");
            Func = Console.ReadLine();

            Console.WriteLine("Введите столбец S:");
            for (int i = 0; i < m; i++)
            {
                cols[i] = decimal.Parse(Console.ReadLine()!);
            }
            Console.WriteLine("Введите массив с кэфами:");
            for (int i = 0; i < m; i++)
            {
                for (int j = 0; j < n; j++)
                {
                    matrix[i, j] = decimal.Parse(Console.ReadLine()!);
                }
            }
            Console.WriteLine("Введите строку F:");
            for (int i = 0; i < n + 1; i++)
            {
                F[i] = decimal.Parse(Console.ReadLine()!);
            }
        }

        // Вывод получившейся симплекс таблицы итоговой
        public void PrintSimplexTab()
        {
            Console.WriteLine();
            Console.WriteLine("Получившаяся симплекс таблица:");
            Console.Write("\t" + "S_i0" + "\t");
            for(int i = 0; i<n; ++i)
            {
                Console.Write(free_variables[i] + "\t");
            }
            Console.WriteLine();
            for(int i = 0; i<m; ++i)
            {
                Console.Write(basis_variables[i] + "\t");
                Console.Write($"{cols_new[i]:F2}" + "\t");
                for(int j = 0; j<n; ++j)
                {
                   Console.Write($"{matrix_new[i,j]:F2}" + "\t");
                }
                Console.WriteLine("");
            }
            Console.Write("F" + "\t");
            for(int i = 0; i<n+1; ++i)
            {
                Console.Write($"{F_new[i]}" + "\t");
            }
            Console.WriteLine($"\n\nF = {F[0]:F2}");
        }

        // Вывод получившейся симплекс таблицы ДЗЛП итоговой
        void PrintSimplexDualTabFinal()
        {
            Console.WriteLine();
            Console.WriteLine("Получившаяся симплекс таблица:\n");
            Console.Write("\t" + "S_io" + "\t");
            for(int i = 0; i<m; ++i)
            {
                Console.Write(free_variables_dual[i] + "\t");
            }
            Console.WriteLine();
            for(int i = 0; i<n; ++i)
            {
                Console.Write(basis_variables_dual[i] + "\t");
                Console.Write(cols_dual_simplex[i] + "\t");
                for(int j = 0; j<m; ++j)
                {
                    Console.Write($"{matrix_dual_simplex[i,j]:F2}" + "\t");
                }
                Console.WriteLine();
            }
            Console.Write("F" + "\t" + $"{cols_dual_simplex[n]:F2}" + "\t");
            for(int i = 0; i<m; ++i)
            {
                Console.Write($"{F_dual_simplex[i]:F2}" + "\t");
            }
            Console.WriteLine($"\n\nF = {cols_dual_simplex[n]:F2}\n");
        }

        // Функция для проверки задачи линейного программирования
        void InputVariables()
        {
            if(TypeLP == "П")
            {
                for(int i = 0; i<n; ++i)
                {
                    free_variables[i] = $"x{i+1}";
                }
                for(int i = 0; i<m; ++i)
                {
                    basis_variables[i] = $"x{n+i+1}";
                }
            }
            else if(TypeLP == "ДЗ")
            {
                for(int i = 0; i<n; ++i)
                {
                    free_variables_dual[i] = $"y{i+1}";
                }
                for(int i = 0; i<m; ++i)
                {
                    basis_variables_dual[i] = $"y{n+i+1}";
                }
            }
        }

        // Функция для замены переменной базисной на свободную и наоборот(подписи)
        void  SwitchVariables(Func<int> find_row, Func<int> find_col)
        {
            int result_row = find_row();
            int result_col = find_col();
            if(result_row == -1 || result_col == -1)
            {
                return;
            }
            if(TypeLP == "П")
            {
                string temp_row = basis_variables[result_row];
                basis_variables[result_row] = free_variables[result_col];
                free_variables[result_col] = temp_row;                
            }
            else if(TypeLP == "ДЗ")
            {
                string temp_row_dual = basis_variables_dual[result_row];
                basis_variables_dual[result_row] = free_variables_dual[result_col];
                free_variables_dual[result_col] = temp_row_dual; 
            }
        }

        // Функция печати таблицы в итерациях
        void PrintSimplexTabWithVariables()
        {
            Console.Write("\t" + "S_io" + "\t");
            for(int i = 0; i < n; ++i)
            {
                Console.Write(free_variables[i] + "\t");
            }
            Console.WriteLine();
            for(int i = 0; i < m; ++i)
            {
                Console.Write(basis_variables[i] + "\t" + $"{cols[i]:F2}" + "\t");
                for(int j = 0; j < n; ++j)
                {
                    Console.Write($"{matrix[i,j]:F2}" + "\t");
                }
                Console.WriteLine();
            }
            Console.Write("F" + "\t");
            for(int i = 0; i < n+1; ++i)
            {
                Console.Write($"{F[i]:F2}" + "\t");
            }
            Console.WriteLine();
            Console.WriteLine();
        }

        // Функция печати симплекс таблицы ДЗЛП в итерациях
        void PrintSimplexDualTab()
        {
            Console.Write("\t" + "S_io" + "\t");
            for(int i = 0; i<m; ++i)
            {
                Console.Write(free_variables_dual[i] + "\t");
            }
            Console.WriteLine();
            for(int i = 0; i<n; ++i)
            {
                Console.Write(basis_variables_dual[i] + "\t");
                Console.Write(cols_dual_simplex[i] + "\t");
                for(int j = 0; j<m; ++j)
                {
                    Console.Write($"{matrix_dual_simplex[i,j]:F2}" + "\t");
                }
                Console.WriteLine();
            }
            Console.Write("F" + "\t" + cols_dual_simplex[n] + "\t");
            for(int i = 0; i < m; ++i)
            {
                Console.Write($"{F_dual_simplex[i]:F2}" + "\t");
            }
            Console.WriteLine();
            Console.WriteLine();
        }

        // Транспонирование матрицы
        void MatrixTransposition()
        {
            for(int i = 0; i<m; ++i)
            {
                F_dual_simplex[i] = cols[i];
            }
            for(int i = 0; i<n+1; ++i)
            {
                cols_dual_simplex[i] = F[i];
            }
            for(int i = 0; i<n; ++i)
            {
                for(int j = 0; j<m; ++j)
                {
                    matrix_dual_simplex[i,j] = matrix[j,i];
                }
            }
        }

        // Поиск разщешающего столбца
        int Find_col()
        {   
            if(TypeLP == "П")
            {
                for(int j = 0; j<m; ++j)
                {
                    if(cols[j] < 0)
                    {
                        for (int i = 0; i < n; i++)
                        {
                            if (matrix[j, i] < 0)
                            {
                                return i;
                            }
                        } 
                    }

                }
            }
            else if(TypeLP == "ДЗ")
            {
                for(int j = 0; j<n; ++j)
                {
                    if(cols_dual_simplex[j] < 0)
                    {
                        for (int i = 0; i < m; i++)
                        {
                            if (matrix_dual_simplex[j,i] < 0)
                            {
                                return i;
                            }
                        } 
                    }

                }
            }
            return -1;
        }

        // Нахождение отношения для поиска разрешающей строки
        int Find_row()
        {
            decimal a;
            decimal b = 10000000000000000000m;
            int c = -1;
            int col_find = Find_col();
            if(col_find == -1)
            {
                return -1;
            }
            if(TypeLP == "П")
            {
                for(int i = 0; i<m; ++i)
                {
                    if(matrix[i,col_find] != 0)
                    {
                        if(cols[i]/matrix[i,col_find] > 0)
                        {
                            a = cols[i]/matrix[i,col_find];
                            if(a < b)
                            {
                                b = a;
                                c = i;
                            }
                        }  
                    }

                }  
            }
            else if(TypeLP == "ДЗ")
            {
                for(int i = 0; i<n; ++i)
                {
                    if(matrix_dual_simplex[i,col_find] != 0)
                    {
                        if(cols_dual_simplex[i]/matrix_dual_simplex[i,col_find] > 0)
                        {
                            a = cols_dual_simplex[i]/matrix_dual_simplex[i,col_find];
                            if(a < b)
                            {
                                b = a;
                                c = i;
                            }
                        }  
                    }

                }
            }
            return c;
        }

        // Преобразование симплекс-таблицы
        void SimplexTab(Func<int> Find_rowfunc, Func<int> Find_colfunc) 
        {
            if(TypeLP == "П")
            {
                int Find_row1 = Find_rowfunc();
                int Find_col1 = Find_colfunc();
                matrix_new[Find_row1,Find_col1] = 1/matrix[Find_row1,Find_col1];
                for(int i = 0; i<n; ++i)
                {
                    if(i != Find_col1)
                    {
                        matrix_new[Find_row1,i] = matrix[Find_row1,i]/matrix[Find_row1,Find_col1];
                    }
                }
                for(int i = 0; i<m; ++i)
                {
                    if(i != Find_row1)
                    {
                        matrix_new[i,Find_col1] = -matrix[i,Find_col1]/matrix[Find_row1,Find_col1];
                    }
                }
                for(int i = 0; i<m; ++i)
                {
                    if(i != Find_row1)
                    {
                        for(int j = 0; j<n; ++j)
                        {
                            if(j != Find_col1)
                            {
                                matrix_new[i,j] = matrix[i,j] - (matrix[Find_row1,j]*matrix[i,Find_col1])/matrix[Find_row1,Find_col1];
                            }
                        }
                    }
                }
                cols_new[Find_row1] = cols[Find_row1]/matrix[Find_row1,Find_col1];
                for(int i = 0; i<m; ++i)
                {
                    if(i != Find_row1)
                    {
                        cols_new[i] = cols[i] - (matrix[i,Find_col1]*cols[Find_row1])/matrix[Find_row1,Find_col1];
                    }
                }
                F_new[Find_col1+1] = -F[Find_col1+1]/matrix[Find_row1,Find_col1];
                F_new[0] = F[0] - ((F[Find_col1+1]*cols[Find_row1])/matrix[Find_row1,Find_col1]);
                if(Find_col1 == n - 1)
                {
                    int variable = 0;
                    for(int i = 1; i<n+1; ++i)
                    {
                        if(i != Find_col1+1)
                        {
                            F_new[i] = F[i] - (F[Find_col1+1]*matrix[Find_row1,variable])/matrix[Find_row1,Find_col1];
                            ++variable;
                        }
                    }
                }
                else
                {
                    int variable = 1;
                    for(int i = 1; i<n+1; ++i)
                    {
                        if(i != Find_col1+1)
                        {
                            F_new[i] = F[i] - (F[Find_col1+1]*matrix[Find_row1,variable])/matrix[Find_row1,Find_col1];
                            ++variable;
                        }
                    }
                }   
            }
            else if(TypeLP == "ДЗ")
            {
                int Find_row1 = Find_rowfunc();
                int Find_col1 = Find_colfunc();
                matrix_dual_simplex_new[Find_row1,Find_col1] = 1/matrix_dual_simplex[Find_row1,Find_col1];
                for(int i = 0; i<m; ++i)
                {
                    if(i != Find_col1)
                    {
                        matrix_dual_simplex_new[Find_row1,i] = matrix_dual_simplex[Find_row1,i]/matrix_dual_simplex[Find_row1,Find_col1];
                    }
                }
                for(int i = 0; i<n; ++i)
                {
                    if(i != Find_row1)
                    {
                        matrix_dual_simplex_new[i,Find_col1] = -matrix_dual_simplex[i,Find_col1]/matrix_dual_simplex[Find_row1,Find_col1];
                    }
                }
                for(int i = 0; i<n; ++i)
                {
                    if(i != Find_row1)
                    {
                        for(int j = 0; j<m; ++j)
                        {
                            if(j != Find_col1)
                            {
                                matrix_dual_simplex_new[i,j] = matrix_dual_simplex[i,j] - (matrix_dual_simplex[Find_row1,j]*matrix_dual_simplex[i,Find_col1])/matrix_dual_simplex[Find_row1,Find_col1];
                            }
                        }
                    }
                }
                cols_dual_simplex_new[Find_row1] = cols_dual_simplex[Find_row1]/matrix_dual_simplex[Find_row1,Find_col1];
                for(int i = 0; i<n; ++i)
                {
                    if(i != Find_row1)
                    {
                        cols_dual_simplex_new[i] = cols_dual_simplex[i] - (matrix_dual_simplex[i,Find_col1]*cols_dual_simplex[Find_row1])/matrix_dual_simplex[Find_row1,Find_col1];
                    }
                }
                F_dual_simplex_new[Find_col1] = -F_dual_simplex[Find_col1]/matrix_dual_simplex[Find_row1,Find_col1];
                cols_dual_simplex_new[n] = cols_dual_simplex[n] - ((F_dual_simplex[Find_col1]*cols_dual_simplex[Find_row1])/matrix_dual_simplex[Find_row1,Find_col1]);
                for(int i = 0; i<m; ++i)
                {
                    if(i != Find_col1)
                    {

                        F_dual_simplex_new[i] = F_dual_simplex[i] - (F_dual_simplex[Find_col1]*matrix_dual_simplex[Find_row1,i])/matrix_dual_simplex[Find_row1,Find_col1];
                    }
                }
            }
        }

        // Функция для проверки наличия решения
        int CheckSystem()
        {
            if(TypeLP == "П")
            {
                int CountVariable = 0;
                bool IfHasntASolver = true;
                for(int i = 0; i<m; ++i)
                {
                    if(cols[i] < 0)
                    {
                        for(int j = 0; j<n; ++j)
                        {
                            if(matrix[i,j] < 0)
                            {
                                IfHasntASolver = false;
                                return 4; // Найден отрицательный элемент => разрешающий столбец, находим разрешающую строчку и продолжаем
                                //симплекс-метод
                            }
                        }
                    }
                    else
                    {
                    ++CountVariable; 
                    }
                }
                if(CountVariable == m)
                {
                    IfHasntASolver = false;
                    return 5; // Опорное решение только, если все свободные кэфы >= 0
                // Опорное решение найдено, продолжаем симплекс-метод
                }
                if(IfHasntASolver)
                {
                    return 3; // В строке S нет отрицательных элементов => система не имеет решений
                }
            }
            else if(TypeLP == "ДЗ")
            {
                int CountVariable = 0;
                bool IfHasntASolver = true;
                for(int i = 0; i<n; ++i)
                {
                    if(cols_dual_simplex[i] < 0)
                    {
                        for(int j = 0; j<m; ++j)
                        {
                            if(matrix_dual_simplex[i,j] < 0)
                            {
                                IfHasntASolver = false;
                                return 4; // Найден отрицательный элемент => разрешающий столбец, находим разрешающую строчку и продолжаем
                                //симплекс-метод
                            }
                        }
                    }
                    else
                    {
                    ++CountVariable; 
                    }
                }
                if(CountVariable == n)
                {
                    IfHasntASolver = false;
                    return 5; // Опорное решение только, если все свободные кэфы >= 0
                // Опорное решение найдено, продолжаем симплекс-метод
                }
                if(IfHasntASolver)
                {
                    return 3; // В строке S нет отрицательных элементов => система не имеет решений
                }
            }
            return -1;
        }

        // Функция для поиска нового разрешающего столбца
        int Find_col_after_general_solver()
        {
            if(TypeLP == "П")
            {
                for(int i = 1; i<n+1; ++i)
                {
                    if(F[i] > 0)
                    {
                        return --i;
                    }
                }
            }
            else if(TypeLP == "ДЗ")
            {
                for(int i = 0; i<m; ++i)
                {
                    if(F_dual_simplex[i] > 0)
                    {
                        return i;
                    }
                }
            }
            return -1;
        }

        // Функция для поиска новой разрешающей строки
        int Find_row_after_general_solver()
        {
            decimal a;
            decimal c = 10000000000000000000m;
            int b = Find_col_after_general_solver();
            if(b == -1)
            {
                return -1;
            }
            int g = -1;
            if(TypeLP == "П")
            {
                for(int i = 0; i<m; ++i)
                {
                    if (matrix[i,b] != 0)
                    {
                        if(cols[i]/matrix[i,b] > 0)
                        {
                            a = cols[i] / matrix[i,b];
                            if(a<c)
                            {
                                c = a;
                                g = i;
                            }
                        }
                    }
                }
            }
            else if(TypeLP == "ДЗ")
            {
                for(int i = 0; i<n; ++i)
                {
                    if (matrix_dual_simplex[i,b] != 0)
                    {
                        if(cols_dual_simplex[i]/matrix_dual_simplex[i,b] > 0)
                        {
                            a = cols_dual_simplex[i] / matrix_dual_simplex[i,b];
                            if(a<c)
                            {
                                c = a;
                                g = i;
                            }
                        }
                    }
                }
            }
            return g;
        }

        // Функция для перемешивания массивов, чтобы реализовать цикл while в функции main_func_for_oprimization для применения симплекс-метода
        // пока не будет получено опорное решение
        public void Refuel_mass()
        {
            if(TypeLP == "П")
            {
                for(int i = 0; i<m; ++i)
                {
                    cols[i] = cols_new[i];
                    for(int j = 0; j<n; ++j)
                    {
                        matrix[i,j] = matrix_new[i,j];
                    }
                }
                for(int i = 0; i<n+1; ++i)
                {
                    F[i] = F_new[i];
                }
            }
            else if(TypeLP == "ДЗ")
            {
                for(int i = 0; i<n; ++i)
                {
                    for(int j = 0; j<m; ++j)
                    {
                        matrix_dual_simplex[i,j] = matrix_dual_simplex_new[i,j];
                        F_dual_simplex[i] = F_dual_simplex_new[i];
                    }
                }
                for(int i = 0; i<n+1; ++i)
                {
                    cols_dual_simplex[i] = cols_dual_simplex_new[i];
                }
            }
        }

        void PrintAfterGeneralSolver()
        {
            Console.Write("\t");
            Console.Write("S_i0" + "\t");
            for(int i = 1; i<n+1; ++i)
            {
                Console.Write($"x{i}" + "\t");
            }
            Console.WriteLine();
            for(int i = 0; i<m; ++i)
            {
                Console.Write($"x{n+i+1}" + "\t");
                Console.Write(cols[i] + "\t");
                for(int j = 0; j<n; ++j)
                {
                    Console.Write(matrix[i,j] + "\t");
                }
                Console.WriteLine();
            }
            Console.Write("F" + "\t");
            for(int i = 0; i<n+1; ++i)
            {
                Console.Write(F[i] + "\t");
            }
            Console.WriteLine();
            Console.WriteLine();
        }

        // Функция для перемешивания массива с функцией(max,min)
        void ShakeMassiveWithFunction()
        {
            if(TypeLP == "ДЗ")
            {
                cols_dual_simplex[n+1] = -cols_dual_simplex[n+1];
                for(int i = 0; i<m; ++i)
                {
                    F_dual_simplex[i] = -F_dual_simplex[i];
                }
            }
            else if(TypeLP == "П")
            {
                for(int i = 0; i<n+1; ++i)
                {
                    F_new[i] = -F_new[i];
                }
            }
        }

        // Главная функция, которая запускает весь процесс нахождения и оптимизации решения целевой функции
        void main_func_for_oprimization()
        {
            Input();
            MatrixTransposition();
            InputVariables();
            if(TypeLP == "П")
            {
                PrintAfterGeneralSolver();
            }
            else if(TypeLP == "ДЗ")
            {
                PrintSimplexDualTab();
            }
            int check_system = CheckSystem();
            if(check_system == 3)
            {
                Console.WriteLine("Система не имеет решений");
                Environment.Exit(0);
            }
            if(check_system == 5)
            {
                if(Find_col_after_general_solver() == -1 || Find_row_after_general_solver() == -1)
                {
                    Console.WriteLine("Система не имеет решений");
                    Environment.Exit(0);
                }
                bool IsItTrue = true;
                while(IsItTrue)
                {
                    IsItTrue = false;
                    if(TypeLP == "П")
                    {
                        for(int i = 1; i<n+1; ++i)
                        {
                            if(F[i] > 0)
                            {
                                IsItTrue = true;
                                break;
                            }
                        }
                        
                    }
                    else if(TypeLP == "ДЗ")
                    {
                        for(int i = 1; i<n; ++i)
                        {
                            if(F_dual_simplex[i] > 0)
                            {
                                IsItTrue = true;
                                break;
                            }
                        }
                    }
                    if(IsItTrue)
                    {
                        Console.WriteLine(Find_col_after_general_solver() + "\t" + Find_row_after_general_solver());
                        SimplexTab(Find_row_after_general_solver,Find_col_after_general_solver);
                        Refuel_mass();
                        SwitchVariables(Find_row_after_general_solver,Find_col_after_general_solver);
                        if(TypeLP == "П")
                        {
                            PrintSimplexTabWithVariables();
                        }
                        else if(TypeLP == "ДЗ")
                        {
                            PrintSimplexDualTab();
                        }
                    }
                }
            }
            // Находим оптимальное решение пока не будет в строке функции(кроме столбца свободных членов) всех отрицательных кэфов
            if(check_system == 4)
            {
                bool IsItTrue = true;
                while(IsItTrue)
                {
                    IsItTrue = false;
                    if(TypeLP == "П")
                    {
                        for(int i = 0; i<m; ++i)
                        {
                            if(cols[i] < 0)
                            {
                                IsItTrue = true;
                                break;
                            }
                        }
                    }
                    else if(TypeLP == "ДЗ")
                    {
                        for(int i = 0; i<n; ++i)
                        {
                            if(cols_dual_simplex[i] < 0)
                            {
                                IsItTrue = true;
                                break;
                            }
                        }
                    }
                    if(IsItTrue)
                    {
                        SimplexTab(Find_row,Find_col);
                        Refuel_mass();
                        SwitchVariables(Find_row,Find_col);
                        if(TypeLP == "П")
                        {
                            PrintSimplexTabWithVariables();
                        }
                        else if(TypeLP == "ДЗ")
                        {
                            PrintSimplexDualTab();
                        }
                    }
                }
                if(Find_col_after_general_solver() == -1 || Find_row_after_general_solver() == -1)
                {
                    Console.WriteLine("Система не имеет решений");
                    Environment.Exit(0);
                }
                IsItTrue = true;
                while(IsItTrue)
                {
                    IsItTrue = false;
                    if(TypeLP == "П")
                    {
                        for(int i = 1; i<n+1; ++i)
                        {
                            if(F[i] > 0)
                            {
                                IsItTrue = true;
                                break;
                            }
                        }
                    }
                    else if(TypeLP == "ДЗ")
                    {
                        for(int i = 1; i<n+1; ++i)
                        {
                            if(F_dual_simplex[i] > 0)
                            {
                                IsItTrue = true;
                                break;
                            }
                        }
                    }
                    if(IsItTrue)
                    {
                        SimplexTab(Find_row_after_general_solver,Find_col_after_general_solver);
                        Refuel_mass();
                        SwitchVariables(Find_row_after_general_solver,Find_col_after_general_solver);
                        if(TypeLP == "П")
                        {
                            PrintSimplexTabWithVariables();
                        }
                        else if(TypeLP == "ДЗ")
                        {
                            PrintSimplexDualTab();
                        }
                    }
                }
            }
            if(TypeLP == "П")
            {
                if(Func == "min")
                {
                    PrintSimplexTab();  
                }
                else if(Func == "max")
                {
                    ShakeMassiveWithFunction();
                    PrintSimplexTab();
                }
            }
            else if(TypeLP == "ДЗ")
            {
                if(Func == "min")
                {
                    ShakeMassiveWithFunction();
                    PrintSimplexDualTabFinal(); 
                }
                else if(Func == "max")
                {
                    PrintSimplexDualTabFinal();
                }
            }
        }



        static void Main()
        {
            Console.Write("Введите количество строк: ");
            int m = int.Parse(Console.ReadLine()!);

            Console.Write("Введите количество столбцов: ");
            int n = int.Parse(Console.ReadLine()!);

            var simplexMethod = new SimplexMethod(n, m);
            simplexMethod.main_func_for_oprimization();












        }

    }





}
```
