# pingpongscorer
Detect ball and create scoring system of live ping pong/table tennis match.

#include <cstdio>
#include <iostream>
#include <opencv2/objdetect/objdetect.hpp>
#include "opencv2/highgui/highgui.hpp"
#include "opencv2/imgproc/imgproc.hpp"
#include <vector>

//things to do : 
//make trackerbars instant by detecting circular figure, then use appropriate color
//add slow motion + 5sec replay


using namespace cv;
using namespace std;

Mat imgOriginal;
int  Player1 = 0, Player2 = 0;
bool GoodBounce;

//Houghcricle function to detect ball
void HoughMethod(Mat imgThresholded, vector<Vec3f> circles){

	HoughCircles(imgThresholded, circles, CV_HOUGH_GRADIENT, 3, imgThresholded.rows / .001, 256, 25, 0, 25);

	//draw circles
	for (size_t i = 0; i < circles.size(); i++)
	{
		Point center(cvRound(circles[i][0]), cvRound(circles[i][1]));
		int radius = cvRound(circles[i][2]);
		// ball center
		circle(imgOriginal, center, 3, Scalar(0, 255, 0), -1, 8, 0);
		// ball outline
		circle(imgOriginal, center, radius, Scalar(0, 0, 255), 3, 8, 0);

		//detects where the last good bounce was (left or right)
		if (center.x > imgOriginal.cols / 2 && center.y > 360)
		{
			GoodBounce = true;
		}
		if (center.x < imgOriginal.cols / 2 && center.y > 360)
		{
			GoodBounce = false;
		}

		//Detects if Ball is out of range and adds point accordingly
		if (center.x < 20 || center.x > imgOriginal.cols - 20)
		{
			if (GoodBounce)
				Player1 = Player1++;
			else
				Player2 = Player2++;
		}

	}
}


int main(int argc, char** argv)
{
	VideoCapture cap(0); //capture the video from first available webcam
	if (!cap.isOpened())  // if not success, exit program
	{
		cout << "No Webcam Detected" << endl;
		return -1;
	}

	namedWindow("Color Control", CV_WINDOW_AUTOSIZE); //create a window called to change detected color in HSV

	int LowH = 0;
	int HighH = 179;

	int LowS = 0;
	int HighS = 255;

	int LowV = 0;
	int HighV = 255;

	//Create trackbars in "Color Control" window
	cvCreateTrackbar("LowH", "Color Control", &LowH, 179); //Hue (0 - 179)
	cvCreateTrackbar("HighH", "Color Control", &HighH, 179);

	cvCreateTrackbar("LowS", "Color Control", &LowS, 255); //Saturation (0 - 255)
	cvCreateTrackbar("HighS", "Color Control", &HighS, 255);

	cvCreateTrackbar("LowV", "Color Control", &LowV, 255); //Value (0 - 255)
	cvCreateTrackbar("HighV", "Color Control", &HighV, 255);

	while (true)
	{
		Mat imgHSV, imgThresholded;

		bool bSuccess = cap.read(imgOriginal); // read a new frame from video
		if (!bSuccess) //if not success, break loop
		{
			cout << "Cannot detect frames" << endl;
			break;
		}

		cvtColor(imgOriginal, imgHSV, COLOR_BGR2HSV); //Convert the captured frame from BGR to HSV

		inRange(imgHSV, Scalar(LowH, LowS, LowV), Scalar(HighH, HighS, HighV), imgThresholded); //Threshold the image

		//morphological opening (remove small objects in thresholded image)
		erode(imgThresholded, imgThresholded, getStructuringElement(MORPH_ELLIPSE, Size(5, 5)));
		dilate(imgThresholded, imgThresholded, getStructuringElement(MORPH_ELLIPSE, Size(5, 5)));

		//morphological closing (fill small holes in thresholded image)
		dilate(imgThresholded, imgThresholded, getStructuringElement(MORPH_ELLIPSE, Size(5, 5)));
		erode(imgThresholded, imgThresholded, getStructuringElement(MORPH_ELLIPSE, Size(5, 5)));

		GaussianBlur(imgThresholded, imgThresholded, Size(9, 9), 2, 2);


		//BallTracking functions //optical flow// SURF// KALMAN FILTER
		vector<Vec3f> circles;
		HoughMethod(imgThresholded, circles);


		//line to represent ping pong table
		line(imgOriginal, Point(0, imgOriginal.rows*.80), Point(imgOriginal.cols, imgOriginal.rows*.80), Scalar(0, 0, 255), 2, 8);
		//line to divide ping pong table in middle
		line(imgOriginal, Point(imgOriginal.cols / 2, 0), Point(imgOriginal.cols / 2, imgOriginal.rows*.80), Scalar(110, 220, 0), 2, 8);

		//Display Ping Pong Score of players
		char Teststr[100];
		sprintf_s(Teststr, "score: %i : %i ", Player1, Player2);
		putText(imgOriginal, Teststr, Point(0, 30), CV_FONT_NORMAL, 1, Scalar(255, 255, 255), 1, 1);

		imshow("Thresholded Image", imgThresholded); //show the thresholded image
		imshow("Original", imgOriginal); //show the original image


		if (waitKey(30) == 27) //press 'esc' key to reset score
		{
			Player1 = 0;
			Player2 = 0;

		}
	}

	return 0;

}
