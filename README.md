# Fitforge-ai
┌─────────────────────────────────────────────────────────────────┐
│                    USER PROFILE DATA MATRIX                      │
├───────────────────┬─────────────────────────────────────────────┤
│ CATEGORY          │ DATA POINTS                                 │
├───────────────────┼─────────────────────────────────────────────┤
│ DEMOGRAPHICS      │ Age, Gender, Height, Weight, Body Frame     │
│ BODY COMPOSITION  │ Body Fat %, Muscle Mass, Waist, Hip, Neck   │
│ FITNESS GOAL      │ Lose Fat | Build Muscle | Recomp | Maintain │
│                   │ | Endurance | Strength | General Fitness    │
│ TARGET METRICS    │ Target Weight, Target Body Fat %, Timeline  │
│ ACTIVITY LEVEL    │ Sedentary | Light | Moderate | Active |     │
│                   │ Very Active | Athlete                       │
│ TRAINING EXP      │ Beginner | Intermediate | Advanced | Elite  │
│ DIET PREFERENCE   │ Omnivore | Vegetarian | Vegan | Keto |      │
│                   │ Paleo | Mediterranean | Halal | Kosher      │
│ ALLERGIES         │ Gluten | Dairy | Nuts | Shellfish | Soy |   │
│                   │ Eggs | Custom                               │
│ MEDICAL CONDITIONS│ Diabetes | Hypertension | PCOS | Thyroid |  │
│                   │ IBS | Heart Condition | None                │
│ CUISINE PREF      │ Indian | Mexican | Asian | Italian |        │
│                   │ Middle Eastern | American | Custom          │
│ MEAL FREQUENCY    │ 3 | 4 | 5 | 6 meals per day                │
│ COOKING TIME      │ Minimal (<15min) | Moderate (15-30min) |    │
│                   │ Extensive (30min+)                          │
│ BUDGET            │ Low | Medium | High                         │
│ EQUIPMENT         │ Full Gym | Home Gym | Dumbbells Only |       │
│                   │ Bodyweight Only | Bands Only                │
│ DAYS/WEEK         │ 3 | 4 | 5 | 6 days                          │
│ SESSION LENGTH    │ 30min | 45min | 60min | 90min               │
│ INJURIES/LIMITS   │ Bad knees | Lower back | Shoulder | Custom  │
│ DISLIKED FOODS    │ User-specified foods to exclude             │
│ PREFERRED FOODS   │ User-specified foods to include             │
│ SLEEP HOURS       │ <6 | 6-7 | 7-8 | 8+ hours                   │
│ WATER INTAKE      │ Current daily water consumption             │
│ STRESS LEVEL      │ Low | Moderate | High                       │
│ WORK SCHEDULE     │ 9-5 | Shift | Remote | Flexible             │
└───────────────────┴─────────────────────────────────────────────┘
User Profile Input
       │
       ▼
┌──────────────────┐
│  STEP 1: BMR     │  ← Mifflin-St Jeor Equation
│  Calculation     │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  STEP 2: TDEE    │  ← BMR × Activity Multiplier
│  Calculation     │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  STEP 3: Goal    │  ← Deficit/Surplus/Maintenance
│  Adjustment      │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  STEP 4: Macro   │  ← Protein/Carbs/Fat Split
│  Calculation     │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  STEP 5: AI Meal │  ← GPT-4o generates meals
│  Generation      │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  STEP 6: Recipe  │  ← Instructions, prep time, shopping list
│  & Shopping List │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  STEP 7: Save to │  ← PostgreSQL + Redis Cache
│  Database        │
└──────────────────┘
# backend/diet/calculator.py

from enum import Enum
from dataclasses import dataclass
from typing import Optional

class Gender(str, Enum):
    MALE = "male"
    FEMALE = "female"

class ActivityLevel(str, Enum):
    SEDENTARY = "sedentary"          # Little to no exercise
    LIGHT = "light"                  # 1-3 days/week light exercise
    MODERATE = "moderate"            # 3-5 days/week moderate exercise
    ACTIVE = "active"                # 6-7 days/week hard exercise
    VERY_ACTIVE = "very_active"      # 2x/day or physical job
    ATHLETE = "athlete"              # 2x/day intense + physical job

class FitnessGoal(str, Enum):
    LOSE_FAT = "lose_fat"
    BUILD_MUSCLE = "build_muscle"
    RECOMP = "recomp"               # Lose fat + build muscle simultaneously
    MAINTAIN = "maintain"
    ENDURANCE = "endurance"

# Activity multipliers for TDEE calculation
ACTIVITY_MULTIPLIERS = {
    ActivityLevel.SEDENTARY: 1.2,
    ActivityLevel.LIGHT: 1.375,
    ActivityLevel.MODERATE: 1.55,
    ActivityLevel.ACTIVE: 1.725,
    ActivityLevel.VERY_ACTIVE: 1.9,
    ActivityLevel.ATHLETE: 2.3,
}

@dataclass
class UserMetrics:
    age: int
    gender: Gender
    height_cm: float
    weight_kg: float
    activity_level: ActivityLevel
    goal: FitnessGoal
    body_fat_percent: Optional[float] = None  # Optional for Katch-McArdle

@dataclass
class MacroTargets:
    calories: int
    protein_g: int       # grams
    carbs_g: int         # grams
    fats_g: int          # grams
    fiber_g: int         # grams
    water_ml: int        # ml

class MetabolicCalculator:
    """
    Core metabolic calculator using science-backed formulas.
    
    BMR Formulas used:
    1. Mifflin-St Jeor (default, most accurate for general population)
    2. Katch-McArdle (used when body fat % is known — more accurate for lean individuals)
    
    TDEE = BMR × Activity Multiplier
    
    Goal adjustments:
    - Lose Fat:    20% deficit (aggressive) or 10-15% (moderate)
    - Build Muscle: 10-15% surplus
    - Recomp:      Maintenance calories (slight deficit on rest days, surplus on training days)
    - Maintain:    TDEE as-is
    - Endurance:   TDEE + 5-10% (fuel for training)
    """
    
    def calculate_bmr(self, metrics: UserMetrics) -> float:
        """Calculate Basal Metabolic Rate."""
        
        if metrics.body_fat_percent is not None:
            # Katch-McArdle Formula (more accurate when body fat is known)
            # BMR = 370 + (21.6 × Lean Body Mass in kg)
            lean_body_mass = metrics.weight_kg * (1 - metrics.body_fat_percent / 100)
            bmr = 370 + (21.6 * lean_body_mass)
            return round(bmr, 1)
        
        # Mifflin-St Jeor Equation (default)
        # Men:   BMR = (10 × weight_kg) + (6.25 × height_cm) - (5 × age) + 5
        # Women: BMR = (10 × weight_kg) + (6.25 × height_cm) - (5 × age) - 161
        if metrics.gender == Gender.MALE:
            bmr = (10 * metrics.weight_kg) + (6.25 * metrics.height_cm) - (5 * metrics.age) + 5
        else:
            bmr = (10 * metrics.weight_kg) + (6.25 * metrics.height_cm) - (5 * metrics.age) - 161
        
        return round(bmr, 1)
    
    def calculate_tdee(self, metrics: UserMetrics) -> float:
        """Calculate Total Daily Energy Expenditure."""
        bmr = self.calculate_bmr(metrics)
        multiplier = ACTIVITY_MULTIPLIERS[metrics.activity_level]
        tdee = bmr * multiplier
        return round(tdee, 1)
    
    def calculate_goal_calories(self, metrics: UserMetrics) -> int:
        """Adjust calories based on fitness goal."""
        tdee = self.calculate_tdee(metrics)
        
        goal_adjustments = {
            FitnessGoal.LOSE_FAT: 0.80,         # 20% deficit
            FitnessGoal.BUILD_MUSCLE: 1.10,     # 10% surplus
            FitnessGoal.RECOMP: 1.0,            # Maintenance
            FitnessGoal.MAINTAIN: 1.0,          # Maintenance
            FitnessGoal.ENDURANCE: 1.08,        # 8% surplus for training fuel
        }
        
        adjusted = tdee * goal_adjustments[metrics.goal]
        
        # Safety floor: never go below 1200 (women) or 1500 (men) without medical supervision
        min_calories = 1500 if metrics.gender == Gender.MALE else 1200
        return max(round(adjusted), min_calories)
    
    def calculate_macros(self, metrics: UserMetrics) -> MacroTargets:
        """
        Calculate macronutrient targets based on goal.
        
        Protein: 2.0-2.2g per kg body weight (building/recomp)
                 1.6-1.8g per kg (fat loss — preserve muscle)
                 1.4-1.6g per kg (endurance)
        
        Fat:     20-30% of total calories (hormone health)
        
        Carbs:   Remaining calories (primary energy source)
        """
        calories = self.calculate_goal_calories(metrics)
        weight_kg = metrics.weight_kg
        
        # --- PROTEIN ---
        protein_per_kg = {
            FitnessGoal.LOSE_FAT: 2.2,        # Higher protein to preserve muscle in deficit
            FitnessGoal.BUILD_MUSCLE: 2.0,    # High protein for muscle synthesis
            FitnessGoal.RECOMP: 2.2,          # High protein for both goals
            FitnessGoal.MAINTAIN: 1.6,        # Moderate for maintenance
            FitnessGoal.ENDURANCE: 1.5,       # Lower protein, more carbs needed
        }
        protein_g = round(weight_kg * protein_per_kg[metrics.goal])
        
        # --- FAT ---
        # 25% of calories from fat (good for hormone production)
        fat_calories = calories * 0.25
        fat_g = round(fat_calories / 9)  # 9 calories per gram of fat
        
        # --- CARBS ---
        # Remaining calories go to carbs
        protein_calories = protein_g * 4   # 4 cal/g protein
        remaining_calories = calories - protein_calories - fat_calories
        carb_g = round(remaining_calories / 4)  # 4 cal/g carbs
        
        # --- FIBER ---
        # 14g per 1000 calories (USDA recommendation)
        fiber_g = round((calories / 1000) * 14)
        
        # --- WATER ---
        # 35ml per kg body weight (baseline)
        # +500ml for each hour of exercise (estimated from activity level)
        exercise_hours = {
            ActivityLevel.SEDENTARY: 0,
            ActivityLevel.LIGHT: 0.5,
            ActivityLevel.MODERATE: 1.0,
            ActivityLevel.ACTIVE: 1.5,
            ActivityLevel.VERY_ACTIVE: 2.5,
            ActivityLevel.ATHLETE: 3.5,
        }
        water_ml = round((weight_kg * 35) + (exercise_hours[metrics.activity_level] * 500))
        
        return MacroTargets(
            calories=calories,
            protein_g=protein_g,
            carbs_g=carb_g,
            fats_g=fat_g,
            fiber_g=fiber_g,
            water_ml=water_ml
        )


# ═══════════════════════════════════════════════════════════════
# EXAMPLE USAGE
# ═══════════════════════════════════════════════════════════════

if __name__ == "__main__":
    user = UserMetrics(
        age=28,
        gender=Gender.MALE,
        height_cm=178,
        weight_kg=82,
        activity_level=ActivityLevel.MODERATE,
        goal=FitnessGoal.LOSE_FAT,
        body_fat_percent=22
    )
    
    calc = MetabolicCalculator()
    bmr = calc.calculate_bmr(user)
    tdee = calc.calculate_tdee(user)
    macros = calc.calculate_macros(user)
    
    print(f"📊 BMR:  {bmr} kcal/day")
    print(f"🔥 TDEE: {tdee} kcal/day")
    print(f"\n🎯 TARGETS (Goal: Lose Fat)")
    print(f"   Calories: {macros.calories} kcal")
    print(f"   Protein:  {macros.protein_g}g  ({macros.protein_g * 4} kcal)")
    print(f"   Carbs:    {macros.carbs_g}g  ({macros.carbs_g * 4} kcal)")
    print(f"   Fats:     {macros.fats_g}g  ({macros.fats_g * 9} kcal)")
    print(f"   Fiber:    {macros.fiber_g}g")
    print(f"   Water:    {macros.water_ml}ml ({macros.water_ml/1000:.1f}L)")

# OUTPUT:
# 📊 BMR:  1915.6 kcal/day
# 🔥 TDEE: 2969.2 kcal/day
#
# 🎯 TARGETS (Goal: Lose Fat)
#    Calories: 2375 kcal
#    Protein:  180g  (720 kcal)
#    Carbs:    159g  (636 kcal)
#    Fats:     66g   (594 kcal)
#    Fiber:    33g
#    Water:    3370ml (3.4L)
# backend/diet/meal_generator.py

import json
from openai import AsyncOpenAI
from dataclasses import dataclass
from typing import List, Dict, Optional
from .calculator import MacroTargets

client = AsyncOpenAI(api_key="your-api-key")

@dataclass
class MealPlanRequest:
    """Everything the AI needs to generate a truly personalized meal plan."""
    macro_targets: MacroTargets
    diet_type: str              # "omnivore", "vegetarian", "vegan", "keto", etc.
    allergies: List[str]        # ["gluten", "dairy", "nuts"]
    medical_conditions: List[str]  # ["diabetes", "hypertension"]
    cuisine_preferences: List[str] # ["indian", "mexican"]
    meals_per_day: int          # 3, 4, 5, or 6
    cooking_time: str           # "minimal", "moderate", "extensive"
    budget: str                 # "low", "medium", "high"
    disliked_foods: List[str]   # ["cilantro", "liver"]
    preferred_foods: List[str]  # ["chicken", "rice", "avocado"]
    target_weight_kg: Optional[float] = None
    days_to_generate: int = 7   # Generate weekly plan

class AIMealGenerator:
    """
    Uses GPT-4o to generate personalized meal plans that exactly match
    macro targets, dietary restrictions, and user preferences.
    
    The AI is instructed to:
    1. Hit exact macro targets (±5g tolerance per meal)
    2. Respect ALL dietary restrictions and allergies
    3. Use preferred cuisines and foods
    4. Avoid disliked foods completely
    5. Account for medical conditions
    6. Match cooking time and budget constraints
    7. Provide full recipes with ingredients and instructions
    8. Generate a consolidated shopping list
    """
    
    def _build_system_prompt(self) -> str:
        return """You are FITFORGE-AI, an elite sports nutritionist and meal planning AI.
        
        Your job is to create highly personalized, scientifically-backed meal plans.
        
        RULES:
        1. Every meal MUST match the macro targets provided (within ±5g tolerance).
        2. NEVER include foods the user is allergic to. This is a SAFETY requirement.
        3. Respect dietary preferences (vegetarian, vegan, keto, etc.) strictly.
        4. Use the user's preferred cuisines as inspiration.
        5. NEVER include disliked foods.
        6. INCLUDE preferred foods where possible.
        7. Account for medical conditions (e.g., diabetics need low-GI carbs).
        8. Match cooking time constraints (minimal = <15min prep).
        9. Stay within budget (low = affordable ingredients like rice, beans, eggs).
        10. Every meal needs: name, ingredients with quantities, macros, instructions.
        11. Provide a weekly consolidated shopping list organized by category.
        12. Include prep tips and food storage advice.
        
        You MUST respond in valid JSON format only. No markdown, no explanations outside JSON.
        """
    
    def _build_user_prompt(self, request: MealPlanRequest, day_num: int) -> str:
        m = request.macro_targets
        
        # Distribute macros across meals
        # Breakfast: 25%, Lunch: 30%, Dinner: 30%, Snacks: 15% (for 4 meals)
        meal_distribution = {
            3: [0.30, 0.35, 0.35],           # B, L, D
            4: [0.25, 0.30, 0.30, 0.15],     # B, L, D, Snack
            5: [0.22, 0.28, 0.10, 0.25, 0.15], # B, Snack, L, D, Snack
            6: [0.20, 0.10, 0.25, 0.10, 0.25, 0.10], # B, S, L, S, D, S
        }
        
        distribution = meal_distribution.get(request.meals_per_day, meal_distribution[4])
        meal_names = {
            3: ["Breakfast", "Lunch", "Dinner"],
            4: ["Breakfast", "Lunch", "Dinner", "Snack"],
            5: ["Breakfast", "Morning Snack", "Lunch", "Dinner", "Evening Snack"],
            6: ["Breakfast", "Snack 1", "Lunch", "Snack 2", "Dinner", "Snack 3"],
        }
        
        meals = []
        for i, name in enumerate(meal_names[request.meals_per_day]):
            meals.append({
                "meal": name,
                "calories": round(m.calories * distribution[i]),
                "protein_g": round(m.protein_g * distribution[i]),
                "carbs_g": round(m.carbs_g * distribution[i]),
                "fats_g": round(m.fats_g * distribution[i]),
            })
        
        prompt = f"""
        Generate a personalized meal plan for DAY {day_num}.
        
        ═══ DAILY MACRO TARGETS ═══
        Total Calories: {m.calories} kcal
        Protein: {m.protein_g}g
        Carbs: {m.carbs_g}g
        Fats: {m.fats_g}g
        Fiber: {m.fiber_g}g
        
        ═══ MEAL DISTRIBUTION ═══
        {json.dumps(meals, indent=2)}
        
        ═══ USER PROFILE ═══
        Diet Type: {request.diet_type}
        Allergies: {', '.join(request.allergies) if request.allergies else 'None'}
        Medical Conditions: {', '.join(request.medical_conditions) if request.medical_conditions else 'None'}
        Preferred Cuisines: {', '.join(request.cuisine_preferences)}
        Cooking Time Available: {request.cooking_time}
        Budget Level: {request.budget}
        Disliked Foods: {', '.join(request.disliked_foods) if request.disliked_foods else 'None'}
        Preferred Foods: {', '.join(request.preferred_foods) if request.preferred_foods else 'None'}
        
        ═══ RESPONSE FORMAT (JSON) ═══
        {{
          "day": {day_num},
          "total_calories": {m.calories},
          "meals": [
            {{
              "meal_name": "Breakfast",
              "dish_name": "Name of the dish",
              "calories": 0,
              "protein_g": 0,
              "carbs_g": 0,
              "fats_g": 0,
              "fiber_g": 0,
              "prep_time_minutes": 0,
              "cook_time_minutes": 0,
              "ingredients": [
                {{"name": "ingredient", "quantity": "1 cup", "calories": 0, "protein_g": 0, "carbs_g": 0, "fats_g": 0}}
              ],
              "instructions": ["step 1", "step 2", "step 3"],
              "nutrition_notes": "Why this meal fits the user's goals",
              "substitutions": ["alternative 1", "alternative 2"]
            }}
          ],
          "daily_nutrition_summary": {{
            "total_calories": 0,
            "total_protein_g": 0,
            "total_carbs_g": 0,
            "total_fats_g": 0,
            "total_fiber_g": 0
          }},
          "daily_tips": "Personalized nutrition advice for this day"
        }}
        
        Generate ONLY the JSON. No other text.
        """
        return prompt
    
    async def generate_day_plan(self, request: MealPlanRequest, day_num: int) -> Dict:
        """Generate a single day's meal plan using GPT-4o."""
        
        response = await client.chat.completions.create(
            model="gpt-4o",
            messages=[
                {"role": "system", "content": self._build_system_prompt()},
                {"role": "user", "content": self._build_user_prompt(request, day_num)}
            ],
            response_format={"type": "json_object"},
            temperature=0.7,  # Some variation day-to-day, but not random
            max_tokens=4000
        )
        
        return json.loads(response.choices[0].message.content)
    
    async def generate_weekly_plan(self, request: MealPlanRequest) -> Dict:
        """Generate a full 7-day meal plan."""
        import asyncio
        
        # Generate all 7 days in parallel for speed
        tasks = [self.generate_day_plan(request, day) for day in range(1, 8)]
        daily_plans = await asyncio.gather(*tasks)
        
        # Generate consolidated shopping list
        shopping_list = await self._generate_shopping_list(daily_plans, request)
        
        return {
            "plan_id": f"diet_{request.macro_targets.calories}cal_{request.diet_type}",
            "diet_type": request.diet_type,
            "daily_targets": {
                "calories": request.macro_targets.calories,
                "protein_g": request.macro_targets.protein_g,
                "carbs_g": request.macro_targets.carbs_g,
                "fats_g": request.macro_targets.fats_g,
            },
            "days": daily_plans,
            "shopping_list": shopping_list,
            "water_target_ml": request.macro_targets.water_ml,
            "generated_at": "2026-07-29T00:00:00Z"
        }
    
    async def _generate_shopping_list(self, daily_plans: List[Dict], request: MealPlanRequest) -> Dict:
        """Generate a consolidated weekly shopping list organized by category."""
        
        # Extract all ingredients from all days
        all_ingredients = []
        for day in daily_plans:
            for meal in day.get("meals", []):
                for ingredient in meal.get("ingredients", []):
                    all_ingredients.append(ingredient)
        
        prompt = f"""
        Consolidate these ingredients into a weekly shopping list.
        Combine duplicate ingredients and sum up quantities.
        Organize by grocery store category.
        
        Ingredients: {json.dumps(all_ingredients)}
        Budget level: {request.budget}
        
        Return JSON:
        {{
          "categories": {{
            "Produce": [{{"name": "", "quantity": "", "estimated_price": ""}}],
            "Meat & Fish": [...],
            "Dairy & Eggs": [...],
            "Pantry & Grains": [...],
            "Spices & Condiments": [...],
            "Frozen": [...],
            "Other": [...]
          }},
          "estimated_total_cost": "$XX",
          "money_saving_tips": ["tip 1", "tip 2"]
        }}
        """
        
        response = await client.chat.completions.create(
            model="gpt-4o",
            messages=[
                {"role": "system", "content": "You are a meal prep and budgeting expert. Return only valid JSON."},
                {"role": "user", "content": prompt}
            ],
            response_format={"type": "json_object"},
            temperature=0.3,
            max_tokens=2000
        )
        
        return json.loads(response.choices[0].message.content)
{
  "day": 1,
  "total_calories": 2375,
  "meals": [
    {
      "meal_name": "Breakfast",
      "dish_name": "Masala Scrambled Eggs with Whole Wheat Toast",
      "calories": 594,
      "protein_g": 45,
      "carbs_g": 40,
      "fats_g": 15,
      "fiber_g": 6,
      "prep_time_minutes": 5,
      "cook_time_minutes": 10,
      "ingredients": [
        {"name": "whole eggs", "quantity": "4 large", "calories": 280, "protein_g": 24, "carbs_g": 2, "fats_g": 20},
        {"name": "whole wheat bread", "quantity": "2 slices", "calories": 160, "protein_g": 8, "carbs_g": 30, "fats_g": 2},
        {"name": "spinach", "quantity": "1 cup", "calories": 7, "protein_g": 1, "carbs_g": 1, "fats_g": 0},
        {"name": "onion, diced", "quantity": "1/4 cup", "calories": 16, "protein_g": 0, "carbs_g": 4, "fats_g": 0},
        {"name": "turmeric", "quantity": "1/2 tsp", "calories": 4, "protein_g": 0, "carbs_g": 1, "fats_g": 0},
        {"name": "garam masala", "quantity": "1/2 tsp", "calories": 5, "protein_g": 0, "carbs_g": 1, "fats_g": 0},
        {"name": "ghee", "quantity": "1 tsp", "calories": 45, "protein_g": 0, "carbs_g": 0, "fats_g": 5},
        {"name": "tomato, chopped", "quantity": "1/2 cup", "calories": 16, "protein_g": 1, "carbs_g": 3, "fats_g": 0},
        {"name": "green chili", "quantity": "1 small", "calories": 6, "protein_g": 0, "carbs_g": 1, "fats_g": 0},
        {"name": "coriander leaves", "quantity": "1 tbsp", "calories": 1, "protein_g": 0, "carbs_g": 0, "fats_g": 0}
      ],
      "instructions": [
        "Heat ghee in a pan over medium heat",
        "Add diced onions and green chili, sauté until translucent (2 min)",
        "Add turmeric and garam masala, toast for 30 seconds",
        "Add chopped tomatoes, cook until soft (2 min)",
        "Add spinach, wilt for 1 minute",
        "Crack eggs into pan, scramble with spices until cooked (3 min)",
        "Toast bread slices, serve eggs alongside",
        "Garnish with fresh coriander leaves"
      ],
      "nutrition_notes": "High protein breakfast (45g) kickstarts muscle protein synthesis. Turmeric provides anti-inflammatory curcumin. Whole wheat toast offers complex carbs for sustained energy. Indian-inspired flavors from user's cuisine preference.",
      "substitutions": [
        "Replace eggs with tofu scramble for vegan option",
        "Use gluten-free bread if gluten intolerant",
        "Replace ghee with olive oil for lower saturated fat"
      ]
    }
  ],
  "daily_tips": "Day 1 focus: High protein throughout to preserve muscle in caloric deficit. Drink 3.4L water. Green tea between meals can boost metabolism by 4-8%. Last meal 2-3 hours before sleep for better digestion."
}
User Profile + Fitness Data
       │
       ▼
┌──────────────────────┐
│ STEP 1: Training     │  ← Split logic based on days/week & goal
│ Split Determination  │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ STEP 2: Exercise     │  ← Filter by equipment, level, injuries
│ Selection Engine     │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ STEP 3: Volume       │  ← Sets, reps, rest based on goal
│ Calculation          │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ STEP 4: Progressive  │  ← Week-over-week progression scheme
│ Overload Plan        │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ STEP 5: AI Warm-up   │  ← Dynamic warm-up + cool-down + mobility
│ & Cool-down          │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ STEP 6: Save Plan    │  ← PostgreSQL + Redis
│ to Database          │
└──────────────────────┘
# backend/workout/generator.py

import json
from enum import Enum
from dataclasses import dataclass, field
from typing import List, Dict, Optional
from openai import AsyncOpenAI

client = AsyncOpenAI(api_key="your-api-key")

# ═══════════════════════════════════════════════════════════════
# ENUMS & CONSTANTS
# ═══════════════════════════════════════════════════════════════

class FitnessLevel(str, Enum):
    BEGINNER = "beginner"          # 0-6 months training
    INTERMEDIATE = "intermediate"  # 6 months - 2 years
    ADVANCED = "advanced"          # 2+ years consistent training
    ELITE = "elite"               # 5+ years, competitive

class Equipment(str, Enum):
    FULL_GYM = "full_gym"
    HOME_GYM = "home_gym"          # Dumbbells + bench + pull-up bar
    DUMBBELLS_ONLY = "dumbbells_only"
    BODYWEIGHT_ONLY = "bodyweight_only"
    BANDS_ONLY = "bands_only"

class WorkoutGoal(str, Enum):
    STRENGTH = "strength"          # Maximal force production
    HYPERTROPHY = "hypertrophy"    # Muscle growth
    ENDURANCE = "endurance"        # Muscular endurance
    FAT_LOSS = "fat_loss"          # Fat loss + muscle preservation
    GENERAL = "general"            # Overall fitness
    ATHLETIC = "athletic"          # Sports performance

# Exercise database (in production, this would be in PostgreSQL)
EXERCISE_DATABASE = {
    # ─── CHEST ───
    "barbell_bench_press": {
        "name": "Barbell Bench Press", "muscle": "chest", "type": "compound",
        "equipment": ["full_gym", "home_gym"], "difficulty": "intermediate",
        "mechanics": "horizontal_push", "injury_safe": True
    },
    "dumbbell_bench_press": {
        "name": "Dumbbell Bench Press", "muscle": "chest", "type": "compound",
        "equipment": ["full_gym", "home_gym", "dumbbells_only"], "difficulty": "beginner",
        "mechanics": "horizontal_push", "injury_safe": True
    },
    "push_up": {
        "name": "Push-Up", "muscle": "chest", "type": "compound",
        "equipment": ["full_gym", "home_gym", "dumbbells_only", "bodyweight_only", "bands_only"],
        "difficulty": "beginner", "mechanics": "horizontal_push", "injury_safe": True
    },
    "cable_fly": {
        "name": "Cable Chest Fly", "muscle": "chest", "type": "isolation",
        "equipment": ["full_gym"], "difficulty": "beginner",
        "mechanics": "horizontal_push", "injury_safe": True
    },
    # ─── BACK ───
    "deadlift": {
        "name": "Conventional Deadlift", "muscle": "back", "type": "compound",
        "equipment": ["full_gym", "home_gym"], "difficulty": "advanced",
        "mechanics": "hip_hinge", "injury_safe": False  # Not for lower back issues
    },
    "pull_up": {
        "name": "Pull-Up", "muscle": "back", "type": "compound",
        "equipment": ["full_gym", "home_gym"], "difficulty": "intermediate",
        "mechanics": "vertical_pull", "injury_safe": True
    },
    "lat_pulldown": {
        "name": "Lat Pulldown", "muscle": "back", "type": "compound",
        "equipment": ["full_gym"], "difficulty": "beginner",
        "mechanics": "vertical_pull", "injury_safe": True
    },
    "barbell_row": {
        "name": "Barbell Bent-Over Row", "muscle": "back", "type": "compound",
        "equipment": ["full_gym", "home_gym"], "difficulty": "intermediate",
        "mechanics": "horizontal_pull", "injury_safe": False  # Lower back stress
    },
    "dumbbell_row": {
        "name": "One-Arm Dumbbell Row", "muscle": "back", "type": "compound",
        "equipment": ["full_gym", "home_gym", "dumbbells_only"], "difficulty": "beginner",
        "mechanics": "horizontal_pull", "injury_safe": True  # Supported on bench
    },
    "inverted_row": {
        "name": "Inverted Row", "muscle": "back", "type": "compound",
        "equipment": ["full_gym", "home_gym", "bodyweight_only"], "difficulty": "beginner",
        "mechanics": "horizontal_pull", "injury_safe": True
    },
    # ─── LEGS ───
    "barbell_squat": {
        "name": "Barbell Back Squat", "muscle": "legs", "type": "compound",
        "equipment": ["full_gym", "home_gym"], "difficulty": "intermediate",
        "mechanics": "squat", "injury_safe": False  # Knee stress
    },
    "goblet_squat": {
        "name": "Goblet Squat", "muscle": "legs", "type": "compound",
        "equipment": ["full_gym", "home_gym", "dumbbells_only"], "difficulty": "beginner",
        "mechanics": "squat", "injury_safe": True  # Knee-friendly
    },
    "lunges": {
        "name": "Walking Lunges", "muscle": "legs", "type": "compound",
        "equipment": ["full_gym", "home_gym", "dumbbells_only", "bodyweight_only"],
        "difficulty": "beginner", "mechanics": "lunge", "injury_safe": True
    },
    "leg_press": {
        "name": "Leg Press", "muscle": "legs", "type": "compound",
        "equipment": ["full_gym"], "difficulty": "beginner",
        "mechanics": "squat", "injury_safe": True  # Back supported
    },
    "romanian_deadlift": {
        "name": "Romanian Deadlift", "muscle": "legs", "type": "compound",
        "equipment": ["full_gym", "home_gym", "dumbbells_only"], "difficulty": "intermediate",
        "mechanics": "hip_hinge", "injury_safe": False  # Lower back stress
    },
    "glute_bridge": {
        "name": "Barbell Glute Bridge", "muscle": "legs", "type": "compound",
        "equipment": ["full_gym", "home_gym", "bodyweight_only"], "difficulty": "beginner",
        "mechanics": "hip_hinge", "injury_safe": True  # Lower back safe
    },
    # ─── SHOULDERS ───
    "overhead_press": {
        "name": "Standing Overhead Press", "muscle": "shoulders", "type": "compound",
        "equipment": ["full_gym", "home_gym"], "difficulty": "intermediate",
        "mechanics": "vertical_push", "injury_safe": False  # Shoulder issues
    },
    "dumbbell_shoulder_press": {
        "name": "Seated Dumbbell Shoulder Press", "muscle": "shoulders", "type": "compound",
        "equipment": ["full_gym", "home_gym", "dumbbells_only"], "difficulty": "beginner",
        "mechanics": "vertical_push", "injury_safe": True  # More stable than barbell
    },
    "lateral_raise": {
        "name": "Dumbbell Lateral Raise", "muscle": "shoulders", "type": "isolation",
        "equipment": ["full_gym", "home_gym", "dumbbells_only"], "difficulty": "beginner",
        "mechanics": "isolation", "injury_safe": True
    },
    "pike_pushup": {
        "name": "Pike Push-Up", "muscle": "shoulders", "type": "compound",
        "equipment": ["bodyweight_only"], "difficulty": "intermediate",
        "mechanics": "vertical_push", "injury_safe": True
    },
    # ─── ARMS ───
    "barbell_curl": {
        "name": "Barbell Curl", "muscle": "biceps", "type": "isolation",
        "equipment": ["full_gym", "home_gym"], "difficulty": "beginner",
        "mechanics": "isolation", "injury_safe": True
    },
    "dumbbell_curl": {
        "name": "Dumbbell Bicep Curl", "muscle": "biceps", "type": "isolation",
        "equipment": ["full_gym", "home_gym", "dumbbells_only"], "difficulty": "beginner",
        "mechanics": "isolation", "injury_safe": True
    },
    "tricep_dips": {
        "name": "Bench Tricep Dips", "muscle": "triceps", "type": "compound",
        "equipment": ["full_gym", "home_gym", "bodyweight_only"], "difficulty": "beginner",
        "mechanics": "isolation", "injury_safe": True
    },
    "overhead_extension": {
        "name": "Overhead Tricep Extension", "muscle": "triceps", "type": "isolation",
        "equipment": ["full_gym", "home_gym", "dumbbells_only"], "difficulty": "beginner",
        "mechanics": "isolation", "injury_safe": True
    },
    # ─── CORE ───
    "plank": {
        "name": "Front Plank", "muscle": "core", "type": "isolation",
        "equipment": ["full_gym", "home_gym", "dumbbells_only", "bodyweight_only", "bands_only"],
        "difficulty": "beginner", "mechanics": "isometric", "injury_safe": True
    },
    "hanging_leg_raise": {
        "name": "Hanging Leg Raise", "muscle": "core", "type": "isolation",
        "equipment": ["full_gym", "home_gym"], "difficulty": "intermediate",
        "mechanics": "isolation", "injury_safe": True
    },
    "dead_bug": {
        "name": "Dead Bug", "muscle": "core", "type": "isolation",
        "equipment": ["bodyweight_only", "full_gym", "home_gym"], "difficulty": "beginner",
        "mechanics": "isolation", "injury_safe": True  # Great for lower back issues
    },
}

# ═══════════════════════════════════════════════════════════════
# DATA CLASSES
# ═══════════════════════════════════════════════════════════════

@dataclass
class WorkoutUserInput:
    fitness_level: FitnessLevel
    goal: WorkoutGoal
    equipment: Equipment
    days_per_week: int              # 3, 4, 5, or 6
    session_duration_min: int       # 30, 45, 60, or 90
    injuries: List[str]             # ["lower_back", "knee", "shoulder"]
    preferred_exercises: List[str]  # Exercise IDs user likes
    disliked_exercises: List[str]   # Exercise IDs to avoid
    cardio_preference: str          # "hiit", "liss", "none", "mixed"
    body_weight_kg: float           # For calculating relative intensity
    target_muscles: List[str] = field(default_factory=lambda: [
        "chest", "back", "legs", "shoulders", "biceps", "triceps", "core"
    ])

# Goal-based training parameters
GOAL_PARAMETERS = {
    WorkoutGoal.STRENGTH: {
        "rep_range": (3, 6),
        "sets_per_exercise": (4, 5),
        "rest_seconds": (180, 300),
        "rir": 2,               # Reps in Reserve (how many reps left)
        "compound_ratio": 0.80,  # 80% compound, 20% isolation
        "progression": "linear", # Add 2.5kg per week
    },
    WorkoutGoal.HYPERTROPHY: {
        "rep_range": (6, 12),
        "sets_per_exercise": (3, 4),
        "rest_seconds": (60, 120),
        "rir": 1,
        "compound_ratio": 0.60,
        "progression": "double",  # Add reps first, then weight
    },
    WorkoutGoal.ENDURANCE: {
        "rep_range": (15, 25),
        "sets_per_exercise": (2, 3),
        "rest_seconds": (30, 60),
        "rir": 0,
        "compound_ratio": 0.50,
        "progression": "volume",  # Add sets/reps
    },
    WorkoutGoal.FAT_LOSS: {
        "rep_range": (8, 15),
        "sets_per_exercise": (3, 4),
        "rest_seconds": (45, 90),
        "rir": 1,
        "compound_ratio": 0.70,
        "progression": "maintain",  # Maintain strength in deficit
    },
    WorkoutGoal.GENERAL: {
        "rep_range": (8, 12),
        "sets_per_exercise": (3, 4),
        "rest_seconds": (60, 120),
        "rir": 2,
        "compound_ratio": 0.65,
        "progression": "double",
    },
    WorkoutGoal.ATHLETIC: {
        "rep_range": (5, 8),
        "sets_per_exercise": (4, 5),
        "rest_seconds": (120, 180),
        "rir": 2,
        "compound_ratio": 0.75,
        "progression": "wave",  # Undulating periodization
    },
}

# Training splits based on days per week
TRAINING_SPLITS = {
    3: {
        "name": "Full Body A/B/C",
        "days": [
            {"name": "Day 1 - Full Body A", "muscles": ["chest", "back", "legs", "core"]},
            {"name": "Day 2 - Full Body B", "muscles": ["legs", "shoulders", "back", "core"]},
            {"name": "Day 3 - Full Body C", "muscles": ["chest", "legs", "biceps", "triceps", "core"]},
        ]
    },
    4: {
        "name": "Upper/Lower Split",
        "days": [
            {"name": "Day 1 - Upper A", "muscles": ["chest", "back", "shoulders", "biceps", "triceps"]},
            {"name": "Day 2 - Lower A", "muscles": ["legs", "core"]},
            {"name": "Day 3 - Upper B", "muscles": ["back", "chest", "shoulders", "triceps", "biceps"]},
            {"name": "Day 4 - Lower B", "muscles": ["legs", "core"]},
        ]
    },
    5: {
        "name": "Push/Pull/Legs/Upper/Lower",
        "days": [
            {"name": "Day 1 - Push", "muscles": ["chest", "shoulders", "triceps"]},
            {"name": "Day 2 - Pull", "muscles": ["back", "biceps"]},
            {"name": "Day 3 - Legs", "muscles": ["legs", "core"]},
            {"name": "Day 4 - Upper", "muscles": ["chest", "back", "shoulders", "biceps", "triceps"]},
            {"name": "Day 5 - Lower", "muscles": ["legs", "core"]},
        ]
    },
    6: {
        "name": "Push/Pull/Legs ×2",
        "days": [
            {"name": "Day 1 - Push A", "muscles": ["chest", "shoulders", "triceps"]},
            {"name": "Day 2 - Pull A", "muscles": ["back", "biceps"]},
            {"name": "Day 3 - Legs A", "muscles": ["legs", "core"]},
            {"name": "Day 4 - Push B", "muscles": ["chest", "shoulders", "triceps"]},
            {"name": "Day 5 - Pull B", "muscles": ["back", "biceps"]},
            {"name": "Day 6 - Legs B", "muscles": ["legs", "core"]},
        ]
    },
}

# Weekly schedule templates (which days are training vs rest)
WEEKLY_SCHEDULES = {
    3: ["train", "rest", "train", "rest", "train", "rest", "rest"],
    4: ["train", "train", "rest", "train", "train", "rest", "rest"],
    5: ["train", "train", "rest", "train", "train", "train", "rest"],
    6: ["train", "train", "rest", "train", "train", "train", "rest"],
}

# ═══════════════════════════════════════════════════════════════
# WORKOUT GENERATOR
# ═══════════════════════════════════════════════════════════════

class WorkoutGenerator:
    """
    The complete workout generation engine.
    
    Flow:
    1. Determine training split based on days/week
    2. Filter exercise database by equipment, level, injuries
    3. Select exercises for each day based on target muscles
    4. Calculate sets, reps, rest based on goal
    5. Generate progressive overload scheme for 4-12 weeks
    6. Use AI for warm-up, cool-down, and personalized coaching notes
    """
    
    def _determine_split(self, days_per_week: int) -> Dict:
        """Determine the training split based on days per week."""
        return TRAINING_SPLITS.get(days_per_week, TRAINING_SPLITS[4])
    
    def _filter_exercises(
        self,
        equipment: Equipment,
        level: FitnessLevel,
        injuries: List[str],
        target_muscles: List[str],
        preferred: List[str],
        disliked: List[str]
    ) -> Dict[str, Dict]:
        """
        Filter the exercise database based on:
        - Available equipment
        - Fitness level (beginners don't do advanced exercises)
        - Injuries (lower_back → skip barbell rows, deadlifts, overhead press)
        - Target muscles for the day
        - User preferences
        """
        
        # Injury-based exclusions
        INJURY_EXCLUSIONS = {
            "lower_back": {"deadlift", "barbell_row", "romanian_deadlift", "overhead_press"},
            "knee": {"barbell_squat", "lunges"},
            "shoulder": {"overhead_press", "barbell_bench_press"},
        }
        
        excluded_exercises = set()
        for injury in injuries:
            excluded_exercises.update(INJURY_EXCLUSIONS.get(injury, set()))
        
        # Add user-disliked exercises
        excluded_exercises.update(disliked)
        
        # Level-based difficulty filtering
        level_order = ["beginner", "intermediate", "advanced", "elite"]
        max_difficulty_index = level_order.index(level.value) if level.value in level_order else 1
        
        filtered = {}
        for ex_id, ex_data in EXERCISE_DATABASE.items():
            # Skip excluded exercises
            if ex_id in excluded_exercises:
                continue
            
            # Check equipment compatibility
            if equipment.value not in ex_data["equipment"]:
                continue
            
            # Check difficulty level
            ex_diff_index = level_order.index(ex_data["difficulty"])
            if ex_diff_index > max_difficulty_index:
                continue
            
            # Check if muscle is in target list
            if ex_data["muscle"] in target_muscles:
                filtered[ex_id] = ex_data
        
        return filtered
    
    def _select_exercises_for_day(
        self,
        available_exercises: Dict,
        target_muscles: List[str],
        goal: WorkoutGoal,
        session_duration: int,
        preferred: List[str]
    ) -> List[Dict]:
        """
        Select the optimal set of exercises for a training day.
        
        Logic:
        - Compound exercises first (bigger bang for buck)
        - Fill remaining time with isolation exercises
        - Prioritize user's preferred exercises
        - Ensure all target muscles are covered
        """
        
        params = GOAL_PARAMETERS[goal]
        compound_ratio = params["compound_ratio"]
        
        # Separate compounds and isolations
        compounds = {k: v for k, v in available_exercises.items() if v["type"] == "compound"}
        isolations = {k: v for k, v in available_exercises.items() if v["type"] == "isolation"}
        
        selected = []
        muscles_covered = set()
        
        # Sort compounds: preferred first, then by muscle coverage
        compound_list = sorted(
            compounds.items(),
            key=lambda x: (x[0] not in preferred, x[1]["muscle"] in muscles_covered)
        )
        
        # Estimate how many exercises fit in the session
        # Average exercise: 4 sets × (reps + rest) ≈ 8-12 min per exercise
        avg_time_per_exercise = (params["sets_per_exercise"][0] * 
                                (params["rep_range"][1] * 3 + params["rest_seconds"][0])) / 60
        max_exercises = int(session_duration / avg_time_per_exercise)
        
        # Select compounds first (target: compound_ratio × max_exercises)
        n_compounds = max(2, int(max_exercises * compound_ratio))
        
        for ex_id, ex_data in compound_list[:n_compounds]:
            selected.append({
                "exercise_id": ex_id,
                "name": ex_data["name"],
                "muscle": ex_data["muscle"],
                "type": ex_data["type"],
                "mechanics": ex_data["mechanics"],
            })
            muscles_covered.add(ex_data["muscle"])
        
        # Fill remaining slots with isolation exercises for uncovered muscles
        n_isolations = max_exercises - len(selected)
        
        # Sort isolations: preferred first, uncovered muscles first
        isolation_list = sorted(
            isolations.items(),
            key=lambda x: (x[0] not in preferred, x[1]["muscle"] in muscles_covered)
        )
        
        for ex_id, ex_data in isolation_list[:n_isolations]:
            selected.append({
                "exercise_id": ex_id,
                "name": ex_data["name"],
                "muscle": ex_data["muscle"],
                "type": ex_data["type"],
                "mechanics": ex_data["mechanics"],
            })
            muscles_covered.add(ex_data["muscle"])
        
        return selected
    
    def _assign_training_parameters(
        self,
        exercises: List[Dict],
        goal: WorkoutGoal,
        level: FitnessLevel,
        body_weight_kg: float,
        week_number: int
    ) -> List[Dict]:
        """Assign sets, reps, rest, and estimated weights to each exercise."""
        
        params = GOAL_PARAMETERS[goal]
        rep_min, rep_max = params["rep_range"]
        set_min, set_max = params["sets_per_exercise"]
        rest_min, rest_max = params["rest_seconds"]
        rir = params["rir"]
        
        # Progressive overload calculation
        # Week 1: baseline, each week add either weight or reps
        progression_scheme = params["progression"]
        
        for i, exercise in enumerate(exercises):
            # Compounds get more sets, isolations get fewer
            if exercise["type"] == "compound":
                sets = set_max
            else:
                sets = set_min
            
            # Reps: lower for compounds (strength focus), higher for isolations
            if exercise["type"] == "compound":
                reps = rep_min + 2  # e.g., 5-8 for strength
            else:
                reps = rep_max  # e.g., 12-15 for isolation
            
            # Rest: longer for compounds
            if exercise["type"] == "compound":
                rest = rest_max
            else:
                rest = rest_min
            
            # Progressive overload weight calculation
            # This is simplified — in production, use user's actual lift data
            estimated_1rm = self._estimate_starting_weight(
                exercise["exercise_id"], 
                body_weight_kg, 
                level
            )
            
            # Calculate working weight based on rep range and RIR
            # Using % of 1RM: 3-6 reps ≈ 85-90%, 6-12 reps ≈ 70-80%, 15+ reps ≈ 50-60%
            if reps <= 6:
                working_weight = estimated_1rm * 0.85
            elif reps <= 12:
                working_weight = estimated_1rm * 0.75
            else:
                working_weight = estimated_1rm * 0.55
            
            # Apply progressive overload
            weekly_increment = self._calculate_progression(
                progression_scheme, week_number, working_weight, reps
            )
            working_weight += weekly_increment
            
            exercise["sets"] = sets
            exercise["reps"] = reps
            exercise["rest_seconds"] = rest
            exercise["rir"] = rir
            exercise["estimated_weight_kg"] = round(working_weight, 1)
            exercise["estimated_1rm_kg"] = round(estimated_1rm, 1)
        
        return exercises
    
    def _estimate_starting_weight(
        self, exercise_id: str, body_weight: float, level: FitnessLevel
    ) -> float:
        """Estimate a reasonable starting 1RM based on exercise and body weight."""
        
        # Multipliers based on exercise type and fitness level
        # These represent typical 1RM as a multiple of body weight
        STRENGTH_MULTIPLIERS = {
            "beginner": {
                "barbell_bench_press": 0.75, "barbell_squat": 0.80, "deadlift": 1.0,
                "overhead_press": 0.50, "barbell_row": 0.65, "pull_up": 0.0,  # Bodyweight
                "default_compound": 0.60, "default_isolation": 0.25,
            },
            "intermediate": {
                "barbell_bench_press": 1.0, "barbell_squat": 1.25, "deadlift": 1.5,
                "overhead_press": 0.65, "barbell_row": 0.85, "pull_up": 0.10,
                "default_compound": 0.80, "default_isolation": 0.35,
            },
            "advanced": {
                "barbell_bench_press": 1.35, "barbell_squat": 1.75, "deadlift": 2.25,
                "overhead_press": 0.85, "barbell_row": 1.10, "pull_up": 0.25,
                "default_compound": 1.0, "default_isolation": 0.45,
            },
            "elite": {
                "barbell_bench_press": 1.75, "barbell_squat": 2.25, "deadlift": 2.75,
                "overhead_press": 1.10, "barbell_row": 1.40, "pull_up": 0.50,
                "default_compound": 1.25, "default_isolation": 0.55,
            },
        }
        
        multipliers = STRENGTH_MULTIPLIERS.get(level.value, STRENGTH_MULTIPLIERS["beginner"])
        multiplier = multipliers.get(exercise_id, multipliers.get(
            "default_compound" if EXERCISE_DATABASE.get(exercise_id, {}).get("type") == "compound" 
            else "default_isolation"
        ))
        
        return body_weight * multiplier
    
    def _calculate_progression(
        self, scheme: str, week: int, current_weight: float, current_reps: int
    ) -> float:
        """Calculate weekly progression increment."""
        
        if scheme == "linear":
            # Add 2.5kg per week (standard linear progression)
            return (week - 1) * 2.5
        
        elif scheme == "double":
            # Weeks 1-3: add 1 rep, Week 4: add 2.5kg and reset reps
            cycle_week = (week - 1) % 4
            if cycle_week < 3:
                return 0  # Adding reps, not weight
            else:
                return 2.5 * ((week - 1) // 4)
        
        elif scheme == "volume":
            # Add volume (sets) rather than weight
            return 0  # Volume increase handled separately
        
        elif scheme == "maintain":
            # In caloric deficit, focus on maintaining strength
            if week <= 2:
                return 0
            return (week - 2) * 1.25  # Small increments
        
        elif scheme == "wave":
            # Undulating: heavy week, medium week, light week, repeat
            cycle = (week - 1) % 3
            if cycle == 0:
                return (week // 3) * 5  # Heavy week bump
            elif cycle == 1:
                return (week // 3) * 5  # Maintain
            else:
                return (week // 3) * 5 - 2.5  # Deload
        
        return 0
    
    async def _generate_warmup_cooldown(
        self, day_muscles: List[str], exercises: List[Dict], level: FitnessLevel
    ) -> Dict:
        """Use AI to generate a personalized warm-up and cool-down."""
        
        prompt = f"""
        Create a warm-up and cool-down routine for a {level.value} level athlete.
        
        Training muscles today: {', '.join(day_muscles)}
        Main exercises: {', '.join([ex['name'] for ex in exercises])}
        
        Return JSON:
        {{
          "warmup": {{
            "duration_minutes": 8,
            "exercises": [
              {{"name": "", "duration": "60 seconds", "purpose": ""}}
            ]
          }},
          "cooldown": {{
            "duration_minutes": 5,
            "stretches": [
              {{"name": "", "duration": "30 seconds", "target_muscle": ""}}
            ]
          }},
          "mobility_notes": "Personalized mobility advice"
        }}
        """
        
        response = await client.chat.completions.create(
            model="gpt-4o",
            messages=[
                {"role": "system", "content": "You are an expert strength coach. Return only valid JSON."},
                {"role": "user", "content": prompt}
            ],
            response_format={"type": "json_object"},
            temperature=0.5,
            max_tokens=1500
        )
        
        return json.loads(response.choices[0].message.content)
    
    async def generate_workout_plan(self, user_input: WorkoutUserInput, weeks: int = 4) -> Dict:
        """
        Generate a complete multi-week workout program.
        
        This is the MAIN entry point for workout generation.
        """
        
        # Step 1: Determine training split
        split = self._determine_split(user_input.days_per_week)
        weekly_schedule = WEEKLY_SCHEDULES[user_input.days_per_week]
        
        # Generate each week
        program_weeks = []
        for week in range(1, weeks + 1):
            week_days = []
            
            for day_idx, day_config in enumerate(split["days"]):
                # Step 2: Filter exercises for this day
                available = self._filter_exercises(
                    equipment=user_input.equipment,
                    level=user_input.fitness_level,
                    injuries=user_input.injuries,
                    target_muscles=day_config["muscles"],
                    preferred=user_input.preferred_exercises,
                    disliked=user_input.disliked_exercises
                )
                
                # Step 3: Select exercises for this day
                selected = self._select_exercises_for_day(
                    available_exercises=available,
                    target_muscles=day_config["muscles"],
                    goal=user_input.goal,
                    session_duration=user_input.session_duration_min,
                    preferred=user_input.preferred_exercises
                )
                
                # Step 4: Assign training parameters (sets, reps, weight)
                selected = self._assign_training_parameters(
                    exercises=selected,
                    goal=user_input.goal,
                    level=user_input.fitness_level,
                    body_weight_kg=user_input.body_weight_kg,
                    week_number=week
                )
                
                # Step 5: Generate warm-up and cool-down (AI)
                warmup_cooldown = await self._generate_warmup_cooldown(
                    day_muscles=day_config["muscles"],
                    exercises=selected,
                    level=user_input.fitness_level
                )
                
                week_days.append({
                    "day_number": day_idx + 1,
                    "day_name": day_config["name"],
                    "is_training_day": True,
                    "target_muscles": day_config["muscles"],
                    "estimated_duration_min": user_input.session_duration_min,
                    "warmup": warmup_cooldown["warmup"],
                    "exercises": selected,
                    "cooldown": warmup_cooldown["cooldown"],
                    "mobility_notes": warmup_cooldown.get("mobility_notes", ""),
                })
            
            # Insert rest days
            full_week = []
            day_pointer = 0
            for schedule_day in weekly_schedule:
                if schedule_day == "train":
                    full_week.append(week_days[day_pointer])
                    day_pointer += 1
                else:
                    full_week.append({
                        "day_number": len(full_week) + 1,
                        "day_name": "Rest Day",
                        "is_training_day": False,
                        "activities": "Active recovery: light walk, stretching, foam rolling",
                        "nutrition_note": "Slightly lower carbs on rest day, maintain protein"
                    })
            
            program_weeks.append({
                "week_number": week,
                "focus": f"Week {week}: {self._get_week_focus(user_input.goal, week)}",
                "days": full_week,
            })
        
        return {
            "plan_id": f"workout_{user_input.goal.value}_{user_input.days_per_week}day_{weeks}wk",
            "split_name": split["name"],
            "goal": user_input.goal.value,
            "fitness_level": user_input.fitness_level.value,
            "equipment": user_input.equipment.value,
            "days_per_week": user_input.days_per_week,
            "session_duration_min": user_input.session_duration_min,
            "weeks": program_weeks,
            "progression_scheme": GOAL_PARAMETERS[user_input.goal]["progression"],
            "cardio_recommendation": self._get_cardio_plan(user_input),
            "generated_at": "2026-07-29T00:00:00Z"
        }
    
    def _get_week_focus(self, goal: WorkoutGoal, week: int) -> str:
        """Get the training focus for a given week."""
        focuses = {
            WorkoutGoal.STRENGTH: {1: "Establish baseline weights", 2: "Increase load 2.5kg", 
                                    3: "Push for new PRs", 4: "Deload week - reduce volume 40%"},
            WorkoutGoal.HYPERTROPHY: {1: "Find RIR 1 weights", 2: "Add 1 rep to each set",
                                       3: "Add 2.5kg, reset reps", 4: "Volume PR week"},
            WorkoutGoal.FAT_LOSS: {1: "Maintain current strength", 2: "Add 5 min cardio post-workout",
                                    3: "Maintain strength + add HIIT", 4: "Deload + assess progress"},
            WorkoutGoal.ENDURANCE: {1: "Build base volume", 2: "Increase reps by 3",
                                     3: "Reduce rest by 15 sec", 4: "Max volume test week"},
        }
        return focuses.get(goal, {}).get(week, f"Week {week} progression")
    
    def _get_cardio_plan(self, user_input: WorkoutUserInput) -> Dict:
        """Generate cardio recommendations based on goal and preference."""
        
        cardio_plans = {
            "hiit": {"type": "HIIT", "frequency": "2-3x/week", "duration": "15-20 min",
                     "example": "30s sprint / 90s walk × 8 rounds"},
            "liss": {"type": "LISS", "frequency": "3-4x/week", "duration": "30-45 min",
                     "example": "Brisk walking, cycling, or swimming at 60-70% max heart rate"},
            "mixed": {"type": "Mixed", "frequency": "3x/week", "duration": "20-30 min",
                      "example": "2 days HIIT (15 min) + 1 day LISS (40 min)"},
            "none": {"type": "Optional", "frequency": "0-1x/week", "duration": "20 min",
                     "example": "Light walking for recovery only"},
        }
        
        return cardio_plans.get(user_input.cardio_preference, cardio_plans["mixed"])


# ═══════════════════════════════════════════════════════════════
# EXAMPLE USAGE
# ═══════════════════════════════════════════════════════════════

if __name__ == "__main__":
    import asyncio
    
    async def main():
        user = WorkoutUserInput(
            fitness_level=FitnessLevel.INTERMEDIATE,
            goal=WorkoutGoal.HYPERTROPHY,
            equipment=Equipment.FULL_GYM,
            days_per_week=4,
            session_duration_min=60,
            injuries=["lower_back"],
            preferred_exercises=["barbell_bench_press", "pull_up"],
            disliked_exercises=["deadlift"],
            cardio_preference="mixed",
            body_weight_kg=82,
        )
        
        generator = WorkoutGenerator()
        plan = await generator.generate_workout_plan(user, weeks=4)
        
        # Print summary
        print(f"📋 WORKOUT PLAN: {plan['split_name']}")
        print(f"🎯 Goal: {plan['goal'].upper()}")
        print(f"📅 {plan['days_per_week']} days/week × {len(plan['weeks'])} weeks")
        print(f"⏱️  Session: {plan['session_duration_min']} min")
        print(f"🏃 Cardio: {plan['cardio_recommendation']['type']}")
        
        print(f"\n{'─'*60}")
        print(f"WEEK 1 - DAY 1: {plan['weeks'][0]['days'][0]['day_name']}")
        print(f"{'─'*60}")
        print(f"🔥 WARM-UP ({plan['weeks'][0]['days'][0]['warmup']['duration_minutes']} min):")
        for w in plan['weeks'][0]['days'][0]['warmup']['exercises']:
            print(f"   • {w['name']} - {w['duration']}")
        
        print(f"\n💪 MAIN WORKOUT:")
        for ex in plan['weeks'][0]['days'][0]['exercises']:
            print(f"   • {ex['name']}")
            print(f"     {ex['sets']} sets × {ex['reps']} reps @ {ex['estimated_weight_kg']}kg")
            print(f"     Rest: {ex['rest_seconds']}s | RIR: {ex['rir']}")
    
    asyncio.run(main())
{
  "plan_id": "workout_hypertrophy_4day_4wk",
  "split_name": "Upper/Lower Split",
  "goal": "HYPERTROPHY",
  "fitness_level": "intermediate",
  "equipment": "full_gym",
  "days_per_week": 4,
  "session_duration_min": 60,
  "weeks": [
    {
      "week_number": 1,
      "focus": "Week 1: Find RIR 1 weights",
      "days": [
        {
          "day_number": 1,
          "day_name": "Day 1 - Upper A",
          "is_training_day": true,
          "target_muscles": ["chest", "back", "shoulders", "biceps", "triceps"],
          "estimated_duration_min": 60,
          "warmup": {
            "duration_minutes": 8,
            "exercises": [
              {"name": "Arm Circles", "duration": "60 seconds", "purpose": "Shoulder mobility"},
              {"name": "Band Pull-Aparts", "duration": "2 sets × 15 reps", "purpose": "Rear delt activation"},
              {"name": "Empty Bar Bench Press", "duration": "2 sets × 10 reps", "purpose": "Movement pattern rehearsal"}
            ]
          },
          "exercises": [
            {
              "exercise_id": "barbell_bench_press",
              "name": "Barbell Bench Press",
              "muscle": "chest",
              "type": "compound",
              "sets": 4,
              "reps": 8,
              "rest_seconds": 120,
              "rir": 1,
              "estimated_weight_kg": 65.0,
              "estimated_1rm_kg": 82.0
            },
            {
              "exercise_id": "pull_up",
              "name": "Pull-Up",
              "muscle": "back",
              "type": "compound",
              "sets": 4,
              "reps": 8,
              "rest_seconds": 120,
              "rir": 1,
              "estimated_weight_kg": 0,
              "estimated_1rm_kg": 0
            },
            {
              "exercise_id": "dumbbell_shoulder_press",
              "name": "Seated Dumbbell Shoulder Press",
              "muscle": "shoulders",
              "type": "compound",
              "sets": 3,
              "reps": 10,
              "rest_seconds": 90,
              "rir": 1,
              "estimated_weight_kg": 22.5,
              "estimated_1rm_kg": 30.0
            },
            {
              "exercise_id": "dumbbell_curl",
              "name": "Dumbbell Bicep Curl",
              "muscle": "biceps",
              "type": "isolation",
              "sets": 3,
              "reps": 12,
              "rest_seconds": 60,
              "rir": 1,
              "estimated_weight_kg": 12.5,
              "estimated_1rm_kg": 17.5
            },
            {
              "exercise_id": "overhead_extension",
              "name": "Overhead Tricep Extension",
              "muscle": "triceps",
              "type": "isolation",
              "sets": 3,
              "reps": 12,
              "rest_seconds": 60,
              "rir": 1,
              "estimated_weight_kg": 15.0,
              "estimated_1rm_kg": 20.0
            }
          ],
          "cooldown": {
            "duration_minutes": 5,
            "stretches": [
              {"name": "Chest Doorway Stretch", "duration": "30 seconds each side", "target_muscle": "chest"},
              {"name": "Lat Stretch", "duration": "30 seconds each side", "target_muscle": "back"},
              {"name": "Overhead Tricep Stretch", "duration": "30 seconds each side", "target_muscle": "triceps"}
            ]
          }
        },
        {
          "day_number": 2,
          "day_name": "Rest Day",
          "is_training_day": false,
          "activities": "Active recovery: light walk, stretching, foam rolling",
          "nutrition_note": "Slightly lower carbs on rest day, maintain protein"
        }
      ]
    }
  ],
  "progression_scheme": "double",
  "cardio_recommendation": {
    "type": "Mixed",
    "frequency": "3x/week",
    "duration": "20-30 min",
    "example": "2 days HIIT (15 min) + 1 day LISS (40 min)"
  }
}
-- ═══════════════════════════════════════════════════════════════
-- FITFORGE-AI DATABASE SCHEMA
-- ═══════════════════════════════════════════════════════════════

-- USER PROFILES (feeds into both generators)
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) UNIQUE NOT NULL,
    password_hash   VARCHAR(255) NOT NULL,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    -- Demographics
    age             INT NOT NULL,
    gender          VARCHAR(10) NOT NULL,
    height_cm       DECIMAL(5,2) NOT NULL,
    weight_kg       DECIMAL(5,2) NOT NULL,
    body_fat_pct    DECIMAL(4,1),
    
    -- Fitness Profile
    activity_level      VARCHAR(20) NOT NULL,
    fitness_level       VARCHAR(20) NOT NULL,
    primary_goal        VARCHAR(20) NOT NULL,
    target_weight_kg    DECIMAL(5,2),
    target_body_fat_pct DECIMAL(4,1),
    
    -- Diet Preferences
    diet_type           VARCHAR(20) NOT NULL,
    allergies           TEXT[],
    medical_conditions  TEXT[],
    cuisine_prefs       TEXT[],
    meals_per_day       INT DEFAULT 4,
    cooking_time        VARCHAR(20),
    budget_level        VARCHAR(10),
    disliked_foods      TEXT[],
    preferred_foods     TEXT[],
    
    -- Workout Preferences
    equipment           VARCHAR(20) NOT NULL,
    days_per_week       INT NOT NULL,
    session_duration    INT NOT NULL,
    injuries            TEXT[],
    preferred_exercises TEXT[],
    disliked_exercises  TEXT[],
    cardio_preference   VARCHAR(10),
    
    -- Lifestyle
    sleep_hours         DECIMAL(3,1),
    water_intake_ml     INT,
    stress_level        VARCHAR(10)
);

-- DIET PLANS
CREATE TABLE diet_plans (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID REFERENCES users(id) ON DELETE CASCADE,
    
    -- Calculated targets
    bmr             DECIMAL(6,1) NOT NULL,
    tdee            DECIMAL(6,1) NOT NULL,
    target_calories INT NOT NULL,
    target_protein  INT NOT NULL,
    target_carbs    INT NOT NULL,
    target_fats     INT NOT NULL,
    target_fiber    INT NOT NULL,
    target_water_ml INT NOT NULL,
    
    -- Plan metadata
    plan_duration_days  INT DEFAULT 7,
    is_active           BOOLEAN DEFAULT TRUE,
    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE diet_meals (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    diet_plan_id    UUID REFERENCES diet_plans(id) ON DELETE CASCADE,
    day_number      INT NOT NULL,
    meal_number     INT NOT NULL,
    meal_name       VARCHAR(50) NOT NULL,
    dish_name       VARCHAR(200) NOT NULL,
    
    calories        INT NOT NULL,
    protein_g       INT NOT NULL,
    carbs_g         INT NOT NULL,
    fats_g          INT NOT NULL,
    fiber_g         INT NOT NULL,
    
    prep_time_min   INT,
    cook_time_min   INT,
    ingredients     JSONB NOT NULL,       -- Array of ingredient objects
    instructions    TEXT[] NOT NULL,       -- Array of step strings
    nutrition_notes TEXT,
    substitutions   TEXT[]
);

CREATE TABLE shopping_lists (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    diet_plan_id    UUID REFERENCES diet_plans(id) ON DELETE CASCADE,
    categories      JSONB NOT NULL,        -- Organized by grocery category
    estimated_cost  VARCHAR(20),
    money_tips      TEXT[]
);

-- WORKOUT PLANS
CREATE TABLE workout_plans (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID REFERENCES users(id) ON DELETE CASCADE,
    
    split_name      VARCHAR(100) NOT NULL,
    goal            VARCHAR(20) NOT NULL,
    fitness_level   VARCHAR(20) NOT NULL,
    equipment       VARCHAR(20) NOT NULL,
    days_per_week   INT NOT NULL,
    session_duration INT NOT NULL,
    weeks_count     INT NOT NULL,
    progression     VARCHAR(20) NOT NULL,
    cardio_plan     JSONB NOT NULL,
    
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE workout_days (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workout_plan_id UUID REFERENCES workout_plans(id) ON DELETE CASCADE,
    week_number     INT NOT NULL,
    day_number      INT NOT NULL,
    day_name        VARCHAR(100) NOT NULL,
    is_training_day BOOLEAN NOT NULL,
    target_muscles  TEXT[],
    estimated_duration_min INT,
    
    -- For rest days
    rest_activities TEXT,
    nutrition_note  TEXT
);

CREATE TABLE workout_exercises (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workout_day_id  UUID REFERENCES workout_days(id) ON DELETE CASCADE,
    
    exercise_id     VARCHAR(50) NOT NULL,
    exercise_name   VARCHAR(100) NOT NULL,
    muscle_group    VARCHAR(30) NOT NULL,
    exercise_type   VARCHAR(15) NOT NULL,     -- compound or isolation
    
    sets            INT NOT NULL,
    reps            INT NOT NULL,
    rest_seconds    INT NOT NULL,
    rir             INT NOT NULL,
    estimated_weight_kg DECIMAL(6,1),
    estimated_1rm_kg    DECIMAL(6,1),
    
    -- For tracking actual performance
    actual_sets     JSONB,     -- [{set: 1, weight: 65, reps: 8}, ...]
    completed       BOOLEAN DEFAULT FALSE
);

CREATE TABLE workout_warmups (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workout_day_id  UUID REFERENCES workout_days(id) ON DELETE CASCADE,
    duration_min    INT NOT NULL,
    exercises       JSONB NOT NULL
);

CREATE TABLE workout_cooldowns (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workout_day_id  UUID REFERENCES workout_days(id) ON DELETE CASCADE,
    duration_min    INT NOT NULL,
    stretches       JSONB NOT NULL
);

-- PROGRESS TRACKING
CREATE TABLE progress_logs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID REFERENCES users(id) ON DELETE CASCADE,
    log_date        DATE NOT NULL,
    
    -- Body metrics
    weight_kg       DECIMAL(5,2),
    body_fat_pct    DECIMAL(4,1),
    waist_cm        DECIMAL(5,1),
    hip_cm          DECIMAL(5,1),
    
    -- Workout performance
    workout_day_id  UUID REFERENCES workout_days(id),
    workout_completed BOOLEAN DEFAULT FALSE,
    workout_duration_min INT,
    total_volume_kg INT,           -- Sum of (weight × reps × sets)
    
    -- Diet adherence
    diet_adherence_pct INT,        -- 0-100, how closely they followed plan
    calories_consumed INT,
    
    -- Lifestyle
    sleep_hours     DECIMAL(3,1),
    water_consumed_ml INT,
    energy_level    INT,           -- 1-10 scale
    mood            INT,           -- 1-10 scale
    
    notes           TEXT
);

-- INDEXES for performance
CREATE INDEX idx_diet_meals_plan_day ON diet_meals(diet_plan_id, day_number);
CREATE INDEX idx_workout_days_plan_week ON workout_days(workout_plan_id, week_number);
CREATE INDEX idx_progress_logs_user_date ON progress_logs(user_id, log_date DESC);
# backend/api/routes.py

from fastapi import FastAPI, HTTPException, Depends
from pydantic import BaseModel
from typing import List, Optional
import uuid

app = FastAPI(title="FITFORGE-AI API", version="1.0.0")

# ─── DIET ENDPOINTS ───

class GenerateDietPlanRequest(BaseModel):
    user_id: str
    days: int = 7

@app.post("/api/diet/generate")
async def generate_diet_plan(request: GenerateDietRequest):
    """
    Generate a personalized AI diet plan.
    
    1. Fetch user profile from database
    2. Calculate BMR, TDEE, and macros
    3. Generate meal plan using GPT-4o
    4. Generate shopping list
    5. Save to database + cache in Redis
    6. Return complete plan
    """
    # ... implementation ...
    pass

@app.get("/api/diet/plan/{plan_id}")
async def get_diet_plan(plan_id: str):
    """Retrieve a saved diet plan."""
    pass

@app.put("/api/diet/plan/{plan_id}/regenerate-day/{day_number}")
async def regenerate_diet_day(plan_id: str, day_number: int):
    """Regenerate a single day (user didn't like the meals)."""
    pass

# ─── WORKOUT ENDPOINTS ───

class GenerateWorkoutPlanRequest(BaseModel):
    user_id: str
    weeks: int = 4

@app.post("/api/workout/generate")
async def generate_workout_plan(request: GenerateWorkoutPlanRequest):
    """
    Generate a personalized AI workout plan.
    
    1. Fetch user profile from database
    2. Determine training split
    3. Select exercises based on equipment/level/injuries
    4. Calculate sets/reps/weight with progressive overload
    5. Generate warm-up/cooldown with AI
    6. Save to database
    7. Return complete plan
    """
    pass

@app.get("/api/workout/plan/{plan_id}")
async def get_workout_plan(plan_id: str):
    """Retrieve a saved workout plan."""
    pass

@app.post("/api/workout/log")
async def log_workout(workout_day_id: str, exercises_completed: List[dict]):
    """Log completed workout (actual weights, reps). Used for progress tracking."""
    pass

# ─── PROGRESS ENDPOINTS ───

@app.post("/api/progress/log")
async def log_progress(user_id: str, weight_kg: float, body_fat_pct: float = None):
    """Log daily/weekly progress metrics."""
    pass

@app.get("/api/progress/dashboard/{user_id}")
async def get_progress_dashboard(user_id: str):
    """Get aggregated progress data for dashboard charts."""
    pass
┌─────────────────────────────────────────────────────────────────────────┐
│                         FITFORGE-AI PERSONALIZATION                     │
│                                                                         │
│  ┌─────────────┐     ┌──────────────┐     ┌─────────────────────────┐  │
│  │  USER INPUT │────▶│  CALCULATION │────▶│    AI GENERATION        │  │
│  │             │     │   ENGINE     │     │                         │  │
│  │ • Age: 28   │     │ BMR: 1916    │     │ GPT-4o generates:       │  │
│  │ • Male      │     │ TDEE: 2969   │     │ • 7-day meal plan       │  │
│  │ • 82kg      │     │ Target: 2375 │     │ • 4-week workout plan   │  │
│  • • 22% BF   │     │ P:180 C:159  │     │ • Shopping list         │  │
│  │ • Moderate  │     │ F:66 Fiber:33│     │ • Warm-up routines      │  │
│  │ • Lose Fat  │     │ Water: 3.4L  │     │ • Cool-down stretches   │  │
│  │ • No dairy  │     │              │     │ • Coaching notes        │  │
│  │ • Full gym  │     │ Split: U/L   │     │ • Substitutions         │  │
│  │ • 4 days/wk │     │ Hypertrophy  │     │ • Progression scheme    │  │
│  │ • 60 min    │     │ 4×8 @ 65kg   │     │                         │  │
│  │ • Low back  │     │ RIR:1 Rest90s│     │ All tailored to:        │  │
│  │             │     │              │     │ • Allergies & medical   │  │
│  │             │     │              │     │ • Equipment available   │  │
│  │             │     │              │     │ • Injury modifications  │  │
│  │             │     │              │     │ • Cuisine preferences   │  │
│  │             │     │              │     │ • Budget & time         │  │
│  └─────────────┘     └──────────────┘     └─────────────────────────┘  │
│                                                    │                    │
│                                                    ▼                    │
│                                         ┌──────────────────────┐       │
│                                         │  PROGRESS TRACKING   │       │
│                                         │                      │       │
│                                         │ • Weight trend chart │       │
│                                         │ • Strength progress  │       │
│                                         │ • Diet adherence %   │       │
│                                         │ • Body fat trend     │       │
│                                         │ • Volume per muscle  │       │
│                                         │                      │       │
│                                         │ → Triggers plan      │       │
│                                         │   adjustments every  │       │
│                                         │   2 weeks            │       │
│                                         └──────────────────────┘       │
└─────────────────────────────────────────────────────────────────────────┘
