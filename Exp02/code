import pandas as pd
import matplotlib.pyplot as plt

FILE = "student_grades.csv"

# 1 & 2. Create CSV
pd.DataFrame({
    "Grade": ["F", "E", "D", "C", "B", "A", "S"],
    "Number_of_Students": [3, 4, 5, 8, 9, 7, 4],
}).to_csv(FILE, index=False)

# 3. Read CSV
df = pd.read_csv(FILE)

# 4. Validate
expected = {"F", "E", "D", "C", "B", "A", "S"}
if set(df["Grade"]) != expected or len(df) != 7:
    raise ValueError("All seven grades (F, E, D, C, B, A, S) must be present.")
if df["Number_of_Students"].sum() != 40:
    raise ValueError("Total number of students must be exactly 40.")
print("Validation successful.\n")

# 5. Percentage
df["Percentage"] = (df["Number_of_Students"] / 40 * 100).round(2)

# 9. Display
print(df.to_string(index=False))

# 10. Highest / lowest
hi = df.loc[df["Number_of_Students"].idxmax()]
lo = df.loc[df["Number_of_Students"].idxmin()]
print(f"\nHighest: Grade {hi['Grade']} ({hi['Number_of_Students']} students)")
print(f"Lowest : Grade {lo['Grade']} ({lo['Number_of_Students']} students)")

# 6. Pie chart
plt.figure(figsize=(7, 7))
plt.pie(df["Number_of_Students"], labels=df["Grade"], autopct="%1.1f%%", startangle=90)
plt.title("Percentage Distribution of Students by Grade")
plt.legend(df["Grade"], title="Grades", loc="center left", bbox_to_anchor=(1, 0.5))
plt.tight_layout()
plt.savefig("grade_pie_chart.png", dpi=150)
plt.show()

# 7. Bar chart
plt.figure(figsize=(8, 5))
bars = plt.bar(df["Grade"], df["Number_of_Students"], color="orange",
               label="Number of Students")
for b in bars:
    plt.text(b.get_x() + b.get_width() / 2, b.get_height() + 0.1,
             int(b.get_height()), ha="center", va="bottom")
plt.xlabel("Grades")
plt.ylabel("Number of Students")
plt.title("Number of Students per Grade")
plt.legend()
plt.tight_layout()
plt.savefig("grade_bar_chart.png", dpi=150)
plt.show()

# 11. Saved as grade_pie_chart.png and grade_bar_chart.png
print("\nCharts saved: grade_pie_chart.png, grade_bar_chart.png")
