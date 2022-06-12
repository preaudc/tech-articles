\title{Spark - good practices: some common caveats and solutions}
\author{Christophe Préaud}

\documentclass[12pt]{article}
\usepackage{color}
\usepackage{fancyvrb}
\usepackage{listings}
 
% include the lines below to use a nicer fixed-width font than the default one
\usepackage{fontspec}
\setmonofont[Scale=0.8]{Inconsolata}

\lstset{fancyvrb=true}
\lstset{
	basicstyle=\small\tt,
	keywordstyle=\color{blue},
	identifierstyle=,
	commentstyle=\color{green},
	stringstyle=\color{red},
	showstringspaces=false,
	tabsize=3,
	numbers=left,
	captionpos=b,
	numberstyle=\tiny
	%stepnumber=4
	}

\begin{document}

\maketitle
 
%\section{Some nice Java code}
% 
%\begin{figure}[h]
%\lstset{language=java,caption=Nicely formatted code.,label=lst:nicecode}
%\begin{lstlisting}
%public class Whatever {
%	public static void main(String[] args) {
%		// whatever
%		System.out.println("Hello, world!");
%	}	
%}
%\end{lstlisting}
%\end{figure}

\section{never collect a dataset}
\section{evaluate as little as possible}
From the Spark ScalaDoc
`Operations available on Datasets are divided into transformations and actions. Transformations are the ones that produce new Datasets, and actions are the ones that trigger computation and return results
(...)
Datasets are "lazy", i.e. computations are only triggered when an action is invoked`

In other words, you should limit the number of actions you apply on your Dataset, since each action will trigger a costly computation, while all transformations are lazily evaluated.

Though it is not alway possible, the ideal would be to call an action on your Dataset only once. Nevertheless, avoid calling an action on your Dataset unless it is necessary.

For example, doing a count (which is an action) for log printing should be avoided.

ds is computed twice in the exemple below
%\begin{figure}[h]
%\lstset{language=scala,caption=Nicely formatted code.,label=lst:nicecode}
%\begin{lstlisting}
%val ds = ds.map({ line =>
%    line.copy(desc = s"this is a \\\${line.desc}")
%})
%println("num elements: " + ds.count)
%ds.collect.foreach(println)
%\end{lstlisting}
%\end{figure}
----------------------------------------------------------------------------------------------------
The simplest and preferable solution would be to skip the print, but if it is absolutely necessary you can use accumulators instead
val accum = sc.longAccumulator("num elements in ds")
val ds = ds.map({ line =>
    accum.add(1)
    line.copy(desc = s"this is a \${line.desc}")
})

ds.collect.foreach(println)
println("num elements: " + accum.value)

\section{use built-in functions rather than UDF}
\section{manage wisely the number of partitions}
\section{deactivate unnecessary cache}
\section{always specify schema when reading file (parquet, json or csv) into a DataFrame}
\section{avoid union performance penalties}
\section{prefer select over withColumn}
\section{(Scala/Java) remove extra columns when converting to a Dataset by providing a class}
\section{(Scala) Prefer immutable variables}


We explain in this section how to obtain headings
for the various sections and subsections of our
document.

\subsection{Headings in the `article' Document Style}

In the `article' style, the document may be divided up
into sections, subsections and subsubsections, and each
can be given a title, printed in a boldface font,
simply by issuing the appropriate command.

\end{document}
