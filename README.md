# K401-Box-Office-Prediction-2025

This is a box-office movie revenue prediction project completed as a coursework for a Machine Learning course (K401) in the Institute of Business Administration, University of Dhaka. 

## Objective

The key objective was to predict the opening week aggregated revenue (domestic) for the movie 'Avatar: Fire and Ash' which, during the completion of the project, remained to be released on 19 December, 2025. The project deadline was 15 December, 2025 (EOD), significantly shaping the choice of features taken to predict.

## Steps

1. Extraction: Around 6700 movies were extracted from 1990-2025 to populate the master dataset for the project. 'BoxOfficeMojo' and 'TMDB' were accessed through API and scraping for movie data.

2. Transformation: The master dataset was cleaned and some new features were engineered to reflect various time-based trends (e.g. seasonality) which are not cleanly reflected in raw data.

3. Modelling: Four Gradient Boosting Machines were pit alongside a baseline linear regression model to choose the best option for the prediction. Finally, CatBoost was chosen. Given the Avatar franchise's track record as a blockbuster, two versions of CatBoost were chosen - the first version was trained with a simple RMSE loss function, and the second was trained with a 90th percentile loss function to get closer to outliers.

4. Final Prediction: Taking the average of the first model's prediction (USD 202M) and the second model's prediction (USD 220M), a prediction of USD 211M was made.

## Files

1. The Artifacts folder contains the first model, the features and the prediction.
2. 'Box_Office_Prediction.ipynb' is the key ipynb file containing the entire prediction. In case GitHub fails to render the code preview, download the ipynb (recommended) or (for a short preview) visit: https://nbviewer.org/github/Jawaad512/K401-Box-Office-Prediction-2025/blob/main/Box_Office_Prediction.ipynb
3. 'box_office_dataset_1990-2025_revised.csv' is the master dataset for the project.
4. 'required_libraries.txt' are the libraries needed to be installed prior to running the code in an ipynb supported environment.
