# Problem-Set-1
AI Transcript
Model: Claude ~ Opus 5.5 ; Date: 10/9

Prompt: [Prep] Act as an applied economist who builds reproducible analysis for client deliverables. [Request] I have a household price index built as a weighted Laspeyres over six spending categories, in a list of dicts named household_basket with keys Item, Price_2019, Price_2024 and weight. Write Python that sweeps each category’s weight from 0 to 0.5, holding the others in proportion, recomputes the index, and produces a single chart showing which category’s weight the result is most sensitive to. [Iterate] numpy, pandas and matplotlib only. Reuse my variable names. No deprecated matplotlib arguments. [Mechanism Check] Add comments explaining why renormalising the other weights is necessary, and what it would mean if the index were highly sensitive to a single category. [Evaluate] Tell me which category dominates my result and what I should say to a client about it.

Model Reply: The script uses plain loops throughout, keeps household_basket, and assumes weight is a 2019 expenditure share. If your weights are quantities, the index formula has to change. There’s a comment flagging this at the top of the file.

On the Evaluate step: I can’t tell you which category dominates, because I haven’t seen your prices or weights. If you paste the printed summary table, I’ll check your reading of it. Here is how to read it when you run it.

Steepest line means most sensitive. Each line is exactly linear. Its slope is the gap between that category’s price relative and the weighted average of all the other categories. So the most sensitive category isn’t necessarily the one with the biggest weight. It’s the one whose price moved most differently from the rest of the basket. Something like energy or rent can be very sensitive even at a modest weight.

Sensitive is not the same as dominant. The table reports two separate numbers:

points_per_0.10_weight tells you how much the index would change if that weight were wrong by 10 percentage points. This measures risk from uncertainty about the weights. share_of_change tells you how much of the measured inflation that category actually contributes at your real weights. This measures what drives the result.

A category can top one column and not the other. When you write “dominates,” decide which of the two claims you’re making.

What to tell a client. Something along these lines: “Our index is most sensitive to [category]. Its prices rose [x]% against roughly [y]% for the rest of the basket. Each 10-point shift in its budget share moves the index by about [z] points. A household that spends more than average on [category] has experienced meaningfully higher inflation than our headline figure, and one that spends less has experienced lower inflation. We recommend validating this weight against the household’s actual spending before relying on the number.”

The point is to present it as a statement about whose cost of living the number represents, not as a flaw in the index.

This looks like it’s for PS1. If the problem set’s AI policy limits Claude to critique for this part, check whether generated code is allowed here before you submit it.

Code that was given:

Weight-sensitivity sweep for the household Laspeyres index
Assumes household_basket already exists: a list of dicts with keys
Item, Price_2019, Price_2024, weight
Assumes weight is a base-period (2019) EXPENDITURE SHARE. If your weights are
quantities instead, the index formula below must change to
sum(P2024 * q) / sum(P2019 * q).
import numpy as np import pandas as pd import matplotlib.pyplot as plt

---------------------------------------------------------------
1. Set up the basket
---------------------------------------------------------------
basket_df = pd.DataFrame(household_basket) basket_df["relative"] = basket_df["Price_2024"] / basket_df["Price_2019"]

Normalise so weights sum to exactly 1 (protects against percent entry or rounding).
basket_df["weight"] = basket_df["weight"] / basket_df["weight"].sum()

items = basket_df["Item"].tolist() relatives = basket_df["relative"].to_numpy() base_weights = basket_df["weight"].to_numpy()

def laspeyres_index(relatives, weights): # Expenditure-share Laspeyres: 100 * sum(w_i * P2024_i / P2019_i) return 100 * np.sum(weights * relatives)

baseline_index = laspeyres_index(relatives, base_weights) print(f"Baseline index (2019 = 100): {baseline_index:.2f}")

---------------------------------------------------------------
2. Sweep each category's weight from 0 to 0.5
---------------------------------------------------------------
WHY RENORMALISE THE OTHER WEIGHTS?
A Laspeyres index is a weighted AVERAGE of price relatives, which only holds
if the weights sum to 1. If we raised one weight without lowering the others,
the weights would sum to more than 1 and the index would rise mechanically,
even with no change in any price. That would measure "a bigger basket", not
"a different mix of spending". Scaling the other weights by (1 - w_k) / (1 - w_k_original)
keeps the total at 1 AND keeps the other categories in the same proportions to
each other, so the only thing that changes is the target category's share.
sweep_values = np.linspace(0, 0.5, 51) sweep_results = {} slopes = {}

for k in range(len(items)): other_total = 1 - base_weights[k] if other_total == 0: print(f"Skipping {items[k]}: it holds the entire basket weight.") continue

index_values = []
for w_k in sweep_values:
    new_weights = base_weights.copy()
    for j in range(len(items)):
        if j != k:
            new_weights[j] = base_weights[j] / other_total * (1 - w_k)
    new_weights[k] = w_k
    index_values.append(laspeyres_index(relatives, new_weights))

sweep_results[items[k]] = index_values
# The index is linear in w_k, so the slope is constant: endpoints are enough.
slopes[items[k]] = (index_values[-1] - index_values[0]) / (sweep_values[-1] - sweep_values[0])
---------------------------------------------------------------
3. Summary table
---------------------------------------------------------------
Two different questions, two different numbers:
points_per_0.10_weight -> how much the index moves if this weight is off by 10 pp
(sensitivity to WEIGHT uncertainty)
share_of_change -> how much of the measured inflation this category
contributes at the actual weights (what DRIVES the result)
total_change = np.sum(base_weights * (relatives - 1))

summary_rows = [] for k in range(len(items)): if items[k] not in slopes: continue if total_change != 0: share = base_weights[k] * (relatives[k] - 1) / total_change else: share = np.nan summary_rows.append({ "Item": items[k], "base_weight": base_weights[k], "pct_price_change": 100 * (relatives[k] - 1), "points_per_0.10_weight": slopes[items[k]] * 0.10, "share_of_change": share, })

summary_df = pd.DataFrame(summary_rows) summary_df["abs_sensitivity"] = summary_df["points_per_0.10_weight"].abs() summary_df = summary_df.sort_values("abs_sensitivity", ascending=False) print(summary_df.drop(columns="abs_sensitivity").round(3).to_string(index=False))

most_sensitive = summary_df["Item"].iloc[0]

---------------------------------------------------------------
4. Chart: one line per category, steepest line = most sensitive
---------------------------------------------------------------
WHAT IF THE INDEX IS HIGHLY SENSITIVE TO ONE CATEGORY?
The slope equals 100 * (that category's relative - the weighted average relative
of everything else). A steep line means that category's price moved very
differently from the rest of the basket. Then the headline number depends heavily
on how much this household actually spends there: a small error in that one
weight shifts the result a lot, and households with a different spending mix
would experience noticeably different inflation from the "average" figure.
fig, ax = plt.subplots(figsize=(9, 5.5))

for item in summary_df["Item"]: k = items.index(item) is_top = item == most_sensitive label = f"{item} ({slopes[item] * 0.10:+.2f} pts per +0.10 weight)" line, = ax.plot( sweep_values, sweep_results[item], linewidth=3 if is_top else 1.5, alpha=1.0 if is_top else 0.7, label=label, ) # Mark where the household's actual weight sits on each line ax.scatter([base_weights[k]], [baseline_index], color=line.get_color(), zorder=3, s=30)

ax.axhline(baseline_index, color="grey", linestyle="--", linewidth=1) ax.text(0.5, baseline_index, f" baseline {baseline_index:.1f}", va="bottom", ha="right", color="grey")

ax.set_xlabel("Weight assigned to category (others rescaled proportionally)") ax.set_ylabel("Laspeyres index (2019 = 100)") ax.set_title(f"Index sensitivity to category weights: most sensitive to {most_sensitive}") ax.legend(frameon=False, fontsize=9, loc="best") ax.spines[["top", "right"]].set_visible(False)

fig.tight_layout() plt.show()

Task 4.2: The written finding (10 pts)
250 words at most, to the director. A number in the first sentence, no code, and a confidence level.

Answer: For a single parent who rents and drives to work on the client's starting wage, living costs rose about 29% from 2019 to 2024, roughly 6 points more than the 23% rise in the official consumer price index. The client's suspicion runs the wrong way. For workers like this, the official index understates what they face rather than overstating it. The reason is where the money goes: rent and childcare take a much larger share of this household's budget than of the average household's, and both rose faster than prices overall. Over the past four decades, rent has risen about 0.7 points a year faster than the official index. The client can be fairly confident about the direction and less confident about the exact size. The result barely changes if this household spends more or less on rent. It is most affected by car insurance, the fastest-rising cost: misjudging insurance's share of the budget by a tenth would shift our figure by about 2 points. Our method also assumes the household kept buying the same things, so families who cut back on items that got pricier faced slightly less than 29%. Two limits matter. This describes one type of household, not the whole workforce, and workers who own homes or don't drive will differ. Our prices are also estimates for this household, not a survey of the client's employees.
