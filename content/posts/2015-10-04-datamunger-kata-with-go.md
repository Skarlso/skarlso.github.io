+++
author = "hannibal"
categories = ["Go", "problem solving", "programming"]
date = "2015-10-04"
title = "DataMunger Kata with Go"
url = "/2015/10/04/datamunger-kata-with-go/"

+++

Quickly wrote up the Data Munger code kata in Go.

Next time, I want better abstractions. And a way to select columns based on their header data. For now, this is not bad.

~~~go

package main

import (
	"bufio"
	"fmt"
	"log"
	"math"
	"os"
	"regexp"
	"strconv"
	"strings"
)

//Data which is Data
type Data struct {
	columnName string
	compareOne float64
	compareTwo float64
}

func main() {
	// datas := []Data{WeatherData{}, FootballData{}}
	fmt.Println("Minimum weather data:", GetDataMinimumDiff("weather.dat", , 1, 2))
	fmt.Println("Minimum football data:", GetDataMinimumDiff("football.dat", 1, 6, 7))
}

//GetDataMinimumDiff gathers data from file to fill up Columns.
func GetDataMinimumDiff(filename string, nameColumn int, compareColOne int, compareColTwo int) Data {
	data := Data{}
	minimum := math.MaxFloat64
	readLines := ReadFile(filename)
	for _, value := range readLines {
		valueArrays := strings.Split(value, ",")
		name := valueArrays[nameColumn]
		trimmedFirst, _ := strconv.ParseFloat(valueArrays[compareColOne], 64)
		trimmedSecond, _ := strconv.ParseFloat(valueArrays[compareColTwo], 64)
		diff := trimmedFirst - trimmedSecond
		diff = math.Abs(diff)
		if diff <= minimum {
			minimum = diff
			data.columnName = name
			data.compareOne = trimmedFirst
			data.compareTwo = trimmedSecond
		}
	}
	return data
}

//ReadFile reads lines from a file and gives back a string array which contains the lines.
func ReadFile(fileName string) (fileLines []string) {
	file, err := os.Open(fileName)
	if err != nil {
		log.Fatal(err)
	}
	defer file.Close()

	scanner := bufio.NewScanner(file)
	//Skipping the first line which is the header.
	scanner.Scan()
	for scanner.Scan() {
		line := scanner.Text()
		re := regexp.MustCompile("\\w+")
		lines := re.FindAllString(line, -1)
		if len(lines) >  {
			fileLines = append(fileLines, strings.Join(lines, ","))
		}
	}

	if err := scanner.Err(); err != nil {
		log.Fatal(err)
	}

	return
}
~~~