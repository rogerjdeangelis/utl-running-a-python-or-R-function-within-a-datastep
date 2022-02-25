# utl-running-a-python-or-R-function-within-a-datastep
Have python compute the area of circles with radii 1,2,3, and 4 within a SAS datastep 
     %let pgm=utl-running-a-python-or-R-function-within-a-datastep;

     Have python compute the area of circles with radii 1,2,3, and 4 within a SAS datastep

     Another way to do this;
     https://blogs.sas.com/content/sgf/2019/06/04/using-python-functions-inside-sas-programs/

     I have updated R abd Python  macros, utl_submit_py64_39.sas and utl_submit_r64.sas.
     I changed  pgm=&pgm to pgm=resolve(&pgm);

     Macros on end ( I am esting the chagnges)

     /*                   _
     (_)_ __  _ __  _   _| |_
     | | `_ \| `_ \| | | | __|
     | | | | | |_) | |_| | |_
     |_|_| |_| .__/ \__,_|\__|
             |_|
     */

     data have;
       radius=1;output;
       radius=2;output;
       radius=3;output;
       radius=4;output;
     run;quit;

     /*******************************************************************/
     /*                                                                 */
     /* Up to 40 obs WORK.HAVE total obs=4 24FEB2022:13:36:40           */
     /*                                                                 */
     /* Obs    RADIUS                                                   */
     /*                                                                 */
     /*  1        1                                                     */
     /*  2        2                                                     */
     /*  3        3                                                     */
     /*  4        4                                                     */
     /*                                                                 */
     /*******************************************************************/

     /*           _               _
       ___  _   _| |_ _ __  _   _| |_
      / _ \| | | | __| `_ \| | | | __|
     | (_) | |_| | |_| |_) | |_| | |_
      \___/ \__,_|\__| .__/ \__,_|\__|
                     |_|
     */

     /*******************************************************************/
     /*                                                                 */
     /* Up to 40 obs WORK.AREA_OF_CIRCLE total obs=4 24FEB2022:13:39:27 */
     /*                                                                 */
     /* Obs    RADIUS      AREA                                         */
     /*                                                                 */
     /*  1        1       3.1416                                        */
     /*  2        2      12.5664                                        */
     /*  3        3      28.2743                                        */
     /*  4        4      50.2655                                        */
     /*                                                                 */
     /*******************************************************************/

     /*           _   _
      _ __  _   _| |_| |__   ___  _ __
     | `_ \| | | | __| `_ \ / _ \| `_ \
     | |_) | |_| | |_| | | | (_) | | | |
     | .__/ \__, |\__|_| |_|\___/|_| |_|
     |_|    |___/
     */

     %symdel radius area / nowarn;

     data area_of_circle;

       set have;

       call symputx('radius',radius);

       put radius=;

       * drop down to Python;
       rc=dosubl('%utl_submit_py64_39("
          import pyperclip;
          import math;
          area=math.pi * &radius * &radius;
          print(area);
          pyperclip.copy(area);
          ",return=area);');

       area=symgetn('area');

       drop rc;

     run;quit;

     /*___
     |  _ \
     | |_) |
     |  _ <
     |_| \_\

     */

     %symdel radius area / nowarn;

     data area_of_circle;

       set have;

       call symputx('radius',radius);

       * drop down to R;
       rc=dosubl('%utl_submit_r64("
          area<-pi * &radius * &radius;
          writeClipboard(as.character(area));
         ",return=area);');

       area=symgetn('area');

       drop rc;

     run;quit;

     /*
      _ __ ___   __ _  ___ _ __ ___  ___
     | `_ ` _ \ / _` |/ __| `__/ _ \/ __|
     | | | | | | (_| | (__| | | (_) \__ \
     |_| |_| |_|\__,_|\___|_|  \___/|___/

     */

     %macro utl_submit_py64_39(
           pgmx
          ,resolve=N
          ,return=N  /* name for the macro variable from Python */
          )/des="Semi colon separated set of python commands - drop down to python";

       * delete temporary files;
       %utlfkil(%sysfunc(pathname(work))/py_pgm.py);
       %utlfkil(%sysfunc(pathname(work))/stderr.txt);
       %utlfkil(%sysfunc(pathname(work))/stdout.txt);

       filename py_pgm "%sysfunc(pathname(work))/py_pgm.py" lrecl=32766 recfm=v;
       data _null_;
         length pgm  $32755 cmd $1024;
         file py_pgm ;
         %if %upcase(&resolve)=Y %then %do;
              pgm=resolve(&pgmx);
         %end;
         %else %do;
              pgm=&pgmx;
         %end;
         pgm=resolve(&pgm);
         semi=countc(pgm,";");
           do idx=1 to semi;
             cmd=cats(scan(pgm,idx,";"));
             if cmd=:". " then
                cmd=trim(substr(cmd,2));
              put cmd $char384.;
              putlog cmd ;
           end;
       run;quit;
       %let _loc=%sysfunc(pathname(py_pgm));
       %let _stderr=%sysfunc(pathname(work))/stderr.txt;
       %let _stdout=%sysfunc(pathname(work))/stdout.txt;
       filename rut pipe  "c:\Python39\python.exe &_loc 2> &_stderr";
       data _null_;
         file print;
         infile rut;
         input;
         put _infile_;
       run;
       filename rut clear;
       filename py_pgm clear;

     data _null_;
         file print;
         infile "%sysfunc(pathname(work))/stderr.txt";
         input;
         put _infile_;
       run;
       filename rut clear;
       filename py_pgm clear;

       * use the clipboard to create macro variable;
       %if "&return" ^= "" %then %do;
         filename clp clipbrd ;
         data _null_;
          length txt $200;
          infile clp;
          input;
          putlog "*******  " _infile_;
          call symputx("&return",_infile_,"G");
         run;quit;
       %end;

     %mend utl_submit_py64_39;


     %macro utl_submit_r64(
           pgmx
          ,return=N
          ,resolve=N
          )/des="Semi colon separated set of R commands - drop down to R";
       * write the program to a temporary file;
       filename r_pgm "%sysfunc(pathname(work))/r_pgm.txt" lrecl=32766 recfm=v;
       data _null_;
         length pgm $32756;
         file r_pgm;
         %if %upcase(&resolve)=Y %then %do;
              pgm=resolve(&pgmx);
         %end;
         %else %do;
              pgm=&pgmx;
         %end;
         putlog pgm;
       run;
       %let __loc=%sysfunc(pathname(r_pgm));
       * pipe file through R;
       filename rut pipe "D:\r412\R\R-4.1.2\bin\R.exe --vanilla --quiet --no-save < &__loc";
       data _null_;
         file print;
         infile rut recfm=v lrecl=32756;
         input;
         put _infile_;
         putlog _infile_;
       run;
       filename rut clear;
       filename r_pgm clear;

       * use the clipboard to create macro variable;
       %if %upcase(%substr(&return.,1,1)) ne N %then %do;
         filename clp clipbrd ;
         data _null_;
          length txt $200;
          infile clp;
          input;
          putlog "macro variable &return = " _infile_;
          call symputx("&return.",_infile_,"G");
         run;quit;
       %end;

     %mend utl_submit_r64;
