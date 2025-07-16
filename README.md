import React, { useEffect, useState } from "react";
import html2canvas from "html2canvas";
import jsPDF from "jspdf";

const courses = [
  // ... (misma lista de cursos)
];

export default function MallaEconomia() {
  const [approved, setApproved] = useState([]);
  const [inProgress, setInProgress] = useState([]);

  useEffect(() => {
    const saved = localStorage.getItem("malla-aprobadas");
    const savedProgress = localStorage.getItem("malla-en-curso");
    if (saved) setApproved(JSON.parse(saved));
    if (savedProgress) setInProgress(JSON.parse(savedProgress));
  }, []);

  useEffect(() => {
    localStorage.setItem("malla-aprobadas", JSON.stringify(approved));
    localStorage.setItem("malla-en-curso", JSON.stringify(inProgress));
  }, [approved, inProgress]);

  const toggleCourse = (id) => {
    if (approved.includes(id)) {
      setApproved((prev) => prev.filter((x) => x !== id));
    } else {
      setApproved((prev) => [...prev, id]);
      setInProgress((prev) => prev.filter((x) => x !== id));
    }
  };

  const markInProgress = (id) => {
    if (inProgress.includes(id)) {
      setInProgress((prev) => prev.filter((x) => x !== id));
    } else {
      if (!approved.includes(id)) {
        setInProgress((prev) => [...prev, id]);
      }
    }
  };

  const isUnlocked = (course) => {
    const prerequisitesMet = course.prerequisites.every((p) => approved.includes(p));
    const year = course.year;
    if (year === 1) return prerequisitesMet;
    const prevYearCourses = courses.filter((c) => c.year === year - 1);
    const prevYearApproved = prevYearCourses.filter((c) => approved.includes(c.id)).length;

    if (year === 2) return prerequisitesMet && prevYearApproved >= 4;
    if (year === 3) {
      const firstYearCourses = courses.filter((c) => c.year === 1);
      const firstYearCompleted = firstYearCourses.every((c) => approved.includes(c.id));
      return prerequisitesMet && firstYearCompleted && prevYearApproved >= 4;
    }
    if (year === 4) {
      const secondYearCourses = courses.filter((c) => c.year === 2);
      const secondYearCompleted = secondYearCourses.every((c) => approved.includes(c.id));
      return prerequisitesMet && secondYearCompleted && prevYearApproved >= 4;
    }
    if (year === 5) {
      const thirdYearCourses = courses.filter((c) => c.year === 3);
      const thirdYearApproved = thirdYearCourses.filter((c) => approved.includes(c.id)).length;
      return prerequisitesMet && thirdYearApproved >= 4;
    }
    return false;
  };

  const yearStatus = (year) => {
    const yearCourses = courses.filter((c) => c.year === year);
    const approvedCount = yearCourses.filter((c) => approved.includes(c.id)).length;
    const allApproved = approvedCount === yearCourses.length;
    return `${approvedCount}/${yearCourses.length} aprobadas${allApproved ? " ✅" : ""}`;
  };

  const exportPDF = () => {
    const input = document.getElementById("malla-container");
    html2canvas(input).then((canvas) => {
      const imgData = canvas.toDataURL("image/png");
      const pdf = new jsPDF("p", "mm", "a4");
      const imgProps = pdf.getImageProperties(imgData);
      const pdfWidth = pdf.internal.pageSize.getWidth();
      const pdfHeight = (imgProps.height * pdfWidth) / imgProps.width;
      pdf.addImage(imgData, "PNG", 0, 0, pdfWidth, pdfHeight);
      pdf.save("malla-economia.pdf");
    });
  };

  const overallProgress = () => {
    const total = courses.length;
    const passed = approved.length;
    const percent = Math.round((passed / total) * 100);
    return { passed, total, percent };
  };

  const { passed, total, percent } = overallProgress();

  return (
    <div className="p-4 space-y-6 font-sans bg-gray-50 min-h-screen" id="malla-container">
      <h1 className="text-3xl font-bold text-center text-gray-800 mb-4">Malla Curricular de Economía</h1>

      <div className="flex justify-center gap-4 mb-4">
        <button
          onClick={() => {
            setApproved([]);
            setInProgress([]);
          }}
          className="px-4 py-2 bg-slate-700 text-white rounded-md shadow hover:bg-slate-800"
        >
          Resetear progreso
        </button>
        <button
          onClick={exportPDF}
          className="px-4 py-2 bg-indigo-600 text-white rounded-md shadow hover:bg-indigo-700"
        >
          Exportar PDF
        </button>
      </div>

      <div className="w-full max-w-3xl mx-auto mb-6">
        <div className="text-center text-lg text-slate-800 font-medium mb-1">
          Avance general: {passed}/{total} materias aprobadas ({percent}%)
        </div>
        <div className="w-full h-4 bg-gray-200 rounded-full overflow-hidden">
          <div
            className="h-full bg-emerald-500 transition-all duration-300"
            style={{ width: `${percent}%` }}
          ></div>
        </div>
      </div>

      {[1, 2, 3, 4, 5].map((year) => (
        <div key={year}>
          <h2 className="text-xl font-bold mb-2 text-slate-800 border-b pb-1">Año {year} <span className="text-sm font-normal text-gray-500">({yearStatus(year)})</span></h2>
          <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-4">
            {courses
              .filter((c) => c.year === year)
              .map((course) => {
                const unlocked = isUnlocked(course);
                const completed = approved.includes(course.id);
                const inProg = inProgress.includes(course.id);
                return (
                  <div
                    key={course.id}
                    onClick={() => unlocked && toggleCourse(course.id)}
                    onContextMenu={(e) => {
                      e.preventDefault();
                      if (unlocked) markInProgress(course.id);
                    }}
                    className={`cursor-pointer p-4 border rounded-xl transition duration-200 shadow-sm relative
                      ${completed ? "line-through bg-emerald-100 text-gray-700" : inProg ? "bg-yellow-100 text-gray-800" : "bg-white text-gray-900"}
                      ${unlocked ? "hover:bg-indigo-100" : "opacity-50 cursor-not-allowed"}`}
                  >
                    <h3 className="text-md font-semibold mb-1">{course.name}</h3>
                    {!unlocked && course.prerequisites.length > 0 && (
                      <p className="text-sm text-gray-500 italic">
                        Requiere: {course.prerequisites.map(id => courses.find(c => c.id === id)?.name).join(", ")}
                      </p>
                    )}
                    {inProg && !completed && (
                      <span className="absolute top-1 right-2 text-xs text-yellow-700 font-semibold">En curso</span>
                    )}
                  </div>
                );
              })}
          </div>
        </div>
      ))}
    </div>
  );
}
