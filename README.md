# Machine-Learning-Project

** This is a file meant for a coursera course. It is not really useful to people doing their own projects.

The first section of the file loads in the necessary libraries used in the file.
  ## Load Libraries
  library(data.table)
  library(tidyverse)
  library(caret)
  library(xgboost)
#################################################################################################

This next section loads in the training data and does some basic EDA. The character variables are either 
removed or set to be integer values since xgboost can't handle character values
  ## 
  training <- fread("C:/Users/HP/Documents/Coursera_Stuff/Machine_Learning/pml-training.csv")

  columm_names <- colnames(training)

  head(training)

  unique(training$classe)

  training <- mutate(training,new_window =  ifelse(new_window == 'no',0,1))
  training$classe <- sapply(training$classe, function(x) {if (x == 'A') {0}
                                                                    else if (x == 'B') {1}
                                                                    else if (x == 'C') {2}
                                                                    else if (x == 'D') {3}
                                                                    else if (x == 'E') {4}})
  training$classe <- as.integer(training$classe)
  training <- training %>% select(-c(user_name, cvtd_timestamp, V1))
  training <- training %>% filter(!is.na(classe))
 
 ###########################################################################################
 
 The next part of the code does some basic variable selection. Since many of the variables are mostly
 null values I wanted to elliminate the ones that won't be split on. This is done by make 3 quick modles
 and selecting only variables that meet a certain importance threshold
 
   ### Variable Selection #################################################
  set.seed(112)
  in_train_base <- createDataPartition(y = training$classe, p = 0.7, list=FALSE)
  base_trian_1 <- training[in_train_base,]
  base_test_1 <- training[-in_train_base,]

  set.seed(317)
  in_train_base <- createDataPartition(y = training$classe, p = 0.7, list=FALSE)
  base_trian_2 <- training[in_train_base,]
  base_test_2 <- training[-in_train_base,]

  set.seed(692)
  in_train_base <- createDataPartition(y = training$classe, p = 0.7, list=FALSE)
  base_trian_3 <- training[in_train_base,]
  base_test_3 <- training[-in_train_base,]

  modled_trian_1 <- xgb.DMatrix(as.matrix(select(base_trian_1,-classe)), label = base_trian_1$classe)
  modled_trian_2 <- xgb.DMatrix(as.matrix(select(base_trian_2,-classe)), label = base_trian_2$classe)
  modled_trian_3 <- xgb.DMatrix(as.matrix(select(base_trian_3,-classe)), label = base_trian_3$classe)


  modled_test_1 <- xgb.DMatrix(as.matrix(select(base_test_1,-classe)), label = base_test_1$classe)
  modled_test_2 <- xgb.DMatrix(as.matrix(select(base_test_2,-classe)), label = base_test_2$classe)
  modled_test_3 <- xgb.DMatrix(as.matrix(select(base_test_3,-classe)), label = base_test_3$classe)

  watchlist_1 <- list(train = modled_trian_1, eval = modled_test_1)
  watchlist_2 <- list(train = modled_trian_2, eval = modled_test_2)
  watchlist_3 <- list(train = modled_trian_3, eval = modled_test_3)

  mean_trian_1 <- mean(base_trian_1$classe)
  mean_trian_2 <- mean(base_trian_2$classe)
  mean_trian_3 <- mean(base_trian_3$classe)

  model_1 <- xgb.train(data = modled_trian_1,
                       nrounds = 500,
                       watchlist = watchlist_1,
                       base_score = mean_trian_1,
                       eta = 0.1,
                       early_stopping_rounds = 10,
                       num_class = 5,
                       subsample = 0.5,
                       colsample_bytree = 0.5,
                       min_child_weight = 1,
                       objective = 'multi:softmax',
                       eval_metric = 'merror')
  model_2 <- xgb.train(data = modled_trian_2,
                       nrounds = 500,
                       watchlist = watchlist_2,
                       base_score = mean_trian_2,
                       eta = 0.1,
                       num_class = 5,
                       early_stopping_rounds = 10,
                       subsample = 0.5,
                       colsample_bytree = 0.5,
                       min_child_weight = 1,
                       objective = 'multi:softmax',
                       eval_metric = 'merror')
  model_3 <- xgb.train(data = modled_trian_3,
                       nrounds = 500,
                       watchlist = watchlist_3,
                       base_score = mean_trian_3,
                       eta = 0.1,
                       num_class = 5,
                       early_stopping_rounds = 10,
                       subsample = 0.5,
                       colsample_bytree = 0.5,
                       min_child_weight = 1,
                       objective = 'multi:softmax',
                       eval_metric = 'merror')

  Importance_1 <- xgb.importance(model = model_1)
  Importance_2 <- xgb.importance(model = model_2)
  Importance_3 <- xgb.importance(model = model_3)

  length <- min(length(Importance_1$Feature), length(Importance_2$Feature), length(Importance_3$Feature))

  Importance_1 <- head(Importance_1, length)
  Importance_2 <- head(Importance_2, length)
  Importance_3 <- head(Importance_3, length)

  imp_all <- merge(merge(Importance_1, Importance_2, by = 'Feature'),Importance_3, by = 'Feature')

  threshold_1 <- 0.01 
  threshold_2 <- 0.02

  imp_all <- imp_all %>% mutate(imp_sum = Gain.x + Gain.y + Gain, Avg_Gain = (Gain.x + Gain.y + Gain)/3) %>%
      filter(imp_sum >= threshold_2 | (Gain.x | Gain.y | Gain) >= threshold_1) %>%
      select(Feature, Avg_Gain)
##################################################################################################

The next part of the code uses the full training data with a 10 kfold cross validation to make sure 
models with the selected variables won't overift and to determine the correct n rounds for the 
final predictive model (model_5) which is fit afterward.
  ###################### Run Model With CV #################################
  training_sel <- training %>% select(c(imp_all$Feature, classe))

  mean_trian_4 <- mean(training$classe)


  modled_train_4 <- xgb.DMatrix(as.matrix(select(training_sel,-classe)), label = training_sel$classe)

  model_4 <- xgb.cv(data = modled_train_4,
                    nrounds = 500,
                    base_score = mean_trian_4,
                    nfold = 10,
                    eta = 0.1,
                    early_stopping_rounds = 10,
                    num_class = 5,
                    subsample = 0.5,
                    colsample_bytree = 0.5,
                    min_child_weight = 1,
                    objective = 'multi:softmax',
                    eval_metric = 'merror',
                    prediction = TRUE)



  model_5 <- xgb.train(data = modled_train_4,
                       nrounds = 173,
                       base_score = mean_trian_4,
                       eta = 0.1,
                       num_class = 5,
                       subsample = 0.5,
                       colsample_bytree = 0.5,
                       min_child_weight = 1,
                       objective = 'multi:softmax',
                       eval_metric = 'merror')
################################################################################################

The final part loads in the test data set and makes predictions on it 
     
  ####### Load in Test data and Make Predictions #####################

  test_final <- fread("C:/Users/HP/Documents/Coursera_Stuff/Machine_Learning/pml-testing.csv")

  test_final <- mutate(test_final,new_window =  ifelse(new_window == 'no',0,1))
  test_final <- test_final %>% select(c(model_5[["feature_names"]]))

  modled_test_final <- xgb.DMatrix(as.matrix(test_final))


  predictions <- predict(model_5, modled_test_final) 
